---
tags:
  - web development
  - dotnet
  - asp
  - c#
  - serilog
  - jsnlog
---

# Adding JSNLog to ASP .Net 6 with Serilog

_2022-01-18_

In this post I will go through the steps required to get an optimal install of JSNLog in ASP dotnet 6 projects using Serilog. JSNLog is a fantastic tool for logging uncaught javascript exceptions to the backend's logging system, and can be used for a variety of other things. This config includes batching of messages to reduce calls made to the server, as well as buffering, where helpful debugging logs are only sent if there's a fatal log that's also being sent, to make it easier to diagnose bugs without unnecessary diagnostic logs appearing at all times.

First you need to install `JSNLog` and `Destructurama.JsonNet` via NuGet. After that, make the following changes to these files.

## _ViewImports.cshtml

Add `@addTagHelper "*, jsnlog"` to enable the tag helper.

## Startup.cs

```csharp linenums="1"
using Destructurama;
using JSNLog;

...
public Startup(IConfiguration configuration, IWebHostEnvironment environment)
{
    ...
    Log.Logger = SerilogLogger.CreateSerilogLogger(serilogConfiguration).Destructure.JsonNetTypes().CreateLogger();
    ...
}

...

public void Configure(
    ...,
    ILoggerFactory loggerFactory)
{
    ...
    loggerFactory.AddSerilog();

    JsnlogConfiguration jsnlogConfiguration = new JsnlogConfiguration
    {
        ajaxAppenders = new List<AjaxAppender> {
            new AjaxAppender {
                name = "appender1",
                storeInBufferLevel = "TRACE", // Log messages with severity smaller than TRACE are ignored
                level = "WARN", // Log messages with severity equal or greater than TRACE and lower than WARN are stored in the internal buffer, but not sent to the server
                                // Log messages with severity equal or greater than WARN and lower than FATAL are sent to the server on their own
                sendWithBufferLevel = "FATAL", // Log messages with severity equal or greater than FATAL are sent to the server, along with all messages stored in the internal buffer
                bufferSize = 20, // Stores the last up to 20 debug messages in browser memory,
                batchSize = 20,
                batchTimeout = 2000, // Logs are guaranteed to be sent within this period (in ms), even if the batch size has not been reached yet
                maxBatchSize = 50 // When the server is unreachable and log messages are being stored until it is reachable again, this is the maximum number of messages that will be stored. Cannot be smaller than batchSize
            }
        },
        loggers = new List<Logger> { // Get the loggers to use the new appender
            new Logger {
                appenders = "appender1"
            }
        },
        insertJsnlogInHtmlResponses = false, // There's an outstanding bug setting this to true so the workaround is using the jl-javascript-logger-definitions tag helper in _Layout.cshtml, via the reference in _ViewImports.cshtml
        productionLibraryPath = null // We're using a fallback from the CDN with hashing, so do not use this
    };

    // Must be before UseStaticFiles and UseAuthorization
    app.UseJSNLog(new CustomLoggingAdapter(loggerFactory), jsnlogConfiguration);
    ...
}
```

## _Layout.cshtml

This one has some interesting configuration.

Firstly, using local fallbacks if the CDN versions of the scripts are unavailable, or do not match their sha384 hashes (i.e. have been hacked or changed).

Secondly, it injects some additional information into the logs: client IP address, user ID (your model may differ from my example), and the user agent.

Thirdly, it sets up logging any uncaught JS exceptions, and any exceptions inside promises where no rejection method is provided.

Fourthly, using stacktrace.js it handles getting the relevant stacktrace using sourcemaps.

```html linenums="1"
<script src="https://cdnjs.cloudflare.com/ajax/libs/jsnlog/2.30.0/jsnlog.min.js"
    asp-fallback-src="~/jsnlog.min.js"
    asp-fallback-test="window.JL"
    crossorigin="anonymous"
    integrity="sha384-ANmgu3V8Mc5/Usd/GeIS0xu0spgLKTIqkSMQAgVvV5C2SRp/rICkLLw5XG/u6BQ9">
</script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/stacktrace.js/2.0.2/stacktrace.min.js"
    asp-fallback-src="~/stacktrace.min.js"
    asp-fallback-test="window.StackTrace"
    crossorigin="anonymous"
    integrity="sha384-4PjQM+vlPbdcaPnFOyBIOfqz90Hvhp+QHb3rBMOy78OaxDHw9mnmzjUSNqJkn+W5">
</script>

<jl-javascript-logger-definitions />

<script>
    async function fetchClientIp() {
        try {
            let response = await fetch('https://api.ipify.org?format=json');
            return await response.json();
        } catch (error) {
            JL().fatalException("Failed to get client IP for logging", error);
        }
    }
    fetchClientIp().then(data => {
        const userDetailsForLogging = {
            'clientIp': data.ip,
            'userId': '@Model.UserId',
            'userAgent': navigator.userAgent
        }
        @* Log uncaught JavaScript errors to the serverside log *@
        if (window) {
            window.onerror = function (errorMsg, url, lineNumber, column, errorObj) {
                var callback = function(stackframes) {
                    var stringifiedStack = stackframes.map(function(sf) {
                        return sf.toString();
                    }).join('\n');
                    const msgObj = {
                        'msg': 'Uncaught exception',
                        'sourceMapStack': stringifiedStack,
                        'user': userDetailsForLogging
                    };
                    JL('serverLog').fatalException({
                        msg: JSON.stringify(msgObj),
                        errorMsg: errorMsg, 
                        url: url, 
                        lineNumber: lineNumber, 
                        column: column
                    }, errorObj);
                };
                var errback = function(err) { console.log(err.message); };
                StackTrace.fromError(errorObj).then(callback).catch(errback);
                return false;
            };
        }
        
        @* Log uncaught JavaScript exceptions inside promises where no rejection method is provided *@
        if (typeof window !== 'undefined') {
            window.onunhandledrejection = function (event) {
                var callback = function(stackframes) {
                    var stringifiedStack = stackframes.map(function(sf) {
                        return sf.toString();
                    }).join('\n');
                    
                    const msgObj = {
                        'msg': 'Unhandled promise rejection',
                        'sourceMapStack': stringifiedStack,
                        'user': userDetailsForLogging
                    };
                    JL("onerrorLogger").fatalException({
                        msg: JSON.stringify(msgObj),
                        errorMsg: event.reason ? event.reason.message : null
                    }, event.reason);
                };
                var errback = function(err) { console.log(err.message); };
                StackTrace.fromError(errorObj).then(callback).catch(errback);
                return false;
            };
        }
    });
</script>
```

## CustomLoggingAdapter.cs

```csharp linenums="1"
using JSNLog;
using Newtonsoft.Json;
using System.Text;

namespace YourProjectHere
{
    /// <summary>
    /// This adapter is required to get JavaScript objects logged by JSNLog to appear correctly in Serilog. Regretably it is required 
    /// because of a defficiency in JSNLog that hasn't been fixed in at least 5 years at the time of writing, and depends on slightly 
    /// customised versions of classes and methods taken from that library, included here in this one file for tidiness.
    /// </summary>
    public class CustomLoggingAdapter : ILoggingAdapter
    {
        private readonly ILoggerFactory _loggerFactory;
        public CustomLoggingAdapter(ILoggerFactory loggerFactory)
        {
            _loggerFactory = loggerFactory;
        }
        public void Log(FinalLogData finalLogData)
        {
            ILogger logger = _loggerFactory.CreateLogger(finalLogData.FinalLogger);
            Object message = LogMessageHelpers.DeserializeIfPossible(finalLogData.FinalMessage);
            switch (finalLogData.FinalLevel)
            {
                case Level.TRACE: logger.LogTrace("{@logMessage}", message); break;
                case Level.DEBUG: logger.LogDebug("{@logMessage}", message); break;
                case Level.INFO: logger.LogInformation("{@logMessage}", message); break;
                case Level.WARN: logger.LogWarning("{@logMessage}", message); break;
                case Level.ERROR: logger.LogError("{@logMessage}", message); break;
                case Level.FATAL: logger.LogCritical("{@logMessage}", message); break;
                default:
                    break;
            }
        }
    }
    internal class LogMessageHelpers
    {
        public static T DeserializeJson<T>(string json)
        {
            T result = JsonConvert.DeserializeObject<T>(json);
            return result;
        }
        public static bool IsPotentialJson(string msg)
        {
            string trimmedMsg = msg.Trim();
            return (trimmedMsg.StartsWith("{") && trimmedMsg.EndsWith("}"));
        }
        /// <summary>
        /// Tries to deserialize msg.
        /// If that works, returns the resulting object.
        /// Otherwise returns msg itself (which is a string).
        /// </summary>
        /// <param name="msg"></param>
        /// <returns></returns>
        public static Object DeserializeIfPossible(string msg)
        {
            try
            {
                if (IsPotentialJson(msg))
                {
                    Object result = DeserializeJson<Object>(msg);
                    return result;
                }
            }
            catch
            {
            }
            return msg;
        }
        /// <summary>
        /// Returns true if the msg contains a valid JSON string.
        /// </summary>
        /// <param name="msg"></param>
        /// <returns></returns>
        public static bool IsJsonString(string msg)
        {
            try
            {
                if (IsPotentialJson(msg))
                {
                    // Try to deserialise the msg. If that does not throw an exception,
                    // decide that msg is a good JSON string.
                    DeserializeJson<Dictionary<string, Object>>(msg);
                    return true;
                }
            }
            catch
            {
            }
            return false;
        }
        /// <summary>
        /// Takes a log message and finds out if it contains a valid JSON string.
        /// If so, returns it unchanged.
        /// 
        /// Otherwise, surrounds the string with quotes (") and escapes the string for JavaScript.
        /// </summary>
        /// <returns></returns>
        public static string EnsureValidJson(string msg)
        {
            if (IsJsonString(msg))
            {
                return msg;
            }
            return JavaScriptStringEncode(msg, true);
        }
        public static string JavaScriptStringEncode(string value, bool addDoubleQuotes)
        {
#if NETFRAMEWORK
            return System.Web.HttpUtility.JavaScriptStringEncode(value, addDoubleQuotes);
#else
            // copied from https://github.com/mono/mono/blob/master/mcs/class/System.Web/System.Web/HttpUtility.cs
            if (String.IsNullOrEmpty(value))
                return addDoubleQuotes ? "\"\"" : String.Empty;
            int len = value.Length;
            bool needEncode = false;
            char c;
            for (int i = 0; i < len; i++)
            {
                c = value[i];
                if (c >= 0 && c <= 31 || c == 34 || c == 39 || c == 60 || c == 62 || c == 92)
                {
                    needEncode = true;
                    break;
                }
            }
            if (!needEncode)
                return addDoubleQuotes ? "\"" + value + "\"" : value;
            var sb = new StringBuilder();
            if (addDoubleQuotes)
                sb.Append('"');
            for (int i = 0; i < len; i++)
            {
                c = value[i];
                if (c >= 0 && c <= 7 || c == 11 || c >= 14 && c <= 31 || c == 39 || c == 60 || c == 62)
                    sb.AppendFormat("\\u{0:x4}", (int)c);
                else switch ((int)c)
                    {
                        case 8:
                            sb.Append("\\b");
                            break;
                        case 9:
                            sb.Append("\\t");
                            break;
                        case 10:
                            sb.Append("\\n");
                            break;
                        case 12:
                            sb.Append("\\f");
                            break;
                        case 13:
                            sb.Append("\\r");
                            break;
                        case 34:
                            sb.Append("\\\"");
                            break;
                        case 92:
                            sb.Append("\\\\");
                            break;
                        default:
                            sb.Append(c);
                            break;
                    }
            }
            if (addDoubleQuotes)
                sb.Append('"');
            return sb.ToString();
#endif
        }
    }
}
```
