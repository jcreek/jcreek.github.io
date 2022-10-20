---
tags:
  - web development
  - dotnet
  - c#
  - pattern matching
---

# C# 8 Pattern Matching

_2022-02-15_

This serves as introductory documentation to the pattern matching features introduced in C# 8, which I think are particularly useful.

## Deconstructors & Positional Patterns

Adding a deconstructor to your class can be done as below. It must be named `Deconstruct` and be a `public void`. Any values we want deconstructed can be set as `out` parameters, then populated. Here all properties are being deconstructed, but a subset could be deconstructed if desired.

```csharp linenums="1"
class Student
{
    public string Name { get; set; }
    public int Year { get; set; }
    public Teacher FormTutor { get; set; }

    public void Deconstruct (out string name, out int year, out Teacher formTutor) 
    {
        name = Name;
        year = Year;
        formTutor = FormTutor;
    }
}
```

To use positional parameters, I'll also set up a second model.

```csharp linenums="1"
class Teacher
{
    public string Name { get; set; }
    public string Subject { get; set; }

    public void Deconstruct (out string name, out string subject) 
    {
        name = Name;
        subject = Subject;
    }
}
```

Here is an example positional pattern. The discards (`_`) are used to 'match all'.

```csharp linenums="1"
public static class YourClassHere
{
    public static bool IsInYear9English(Student student)
    {
        return student is Student(
            _,
            9,
            Teacher (_, "English")
        );
    }
}
```

While an if statement would probably be easier to read and maintain in this example, positional patterns enable recursion to be used, which can be very useful.

## Property Patterns

I've updated the Student model to add a School property.

```csharp linenums="1"
class Student
{
    public string Name { get; set; }
    public int Year { get; set; }
    public string School { get; set; }
    public Teacher FormTutor { get; set; }
}
```

I prefer using property patterns to positional patterns as they are much more readable. Here's an example method to check whether a student attends a particular school and has a form tutor who teaches Maths.

```csharp linenums="1"
public static class YourClassHere
{
    public static bool IsStudentInGreenAcademyWithMathsFormTutor(Student student)
    {
        return student is {
            School: "Green Academy", 
            FormTutor: {
                Subject: "Maths"
            } 
        };
    }
}
```

This can be made more generic to accept an `object` rather than a `Student`, and check that object is a Student.

```csharp linenums="1"
public static class YourClassHere
{
    public static bool IsStudentInGreenAcademyWithMathsFormTutor(object obj)
    {
        return obj is Student student && 
            student is {
                School: "Green Academy", 
                FormTutor: {
                    Subject: "Maths"
                } 
            };
    }
}
```

## Switch Expressions

These can be used in place of standard switch cases. I'm using a discard (`_`) for catching the default case for unmatched patterns.

```csharp linenums="1"
public static class YourClassHere
{
    public static string DisplayPersonInfo (object person)
    {
        string result = person switch
        {
            Student student => $"Student at {student.School} in Year {student.Year}",
            Teacher teacher => $"Teacher of {teacher.Subject}",
            _ => "Person is not a student or a teacher"
        };
        
        return result;
    }
}
```

The syntax can make it easier to read, and it can be a lot more powerful. You can also define recursive switch patterns.

```csharp linenums="1"
public static class YourClassHere
{
    public static string DisplayPersonInfo (object person)
    {
        string result = person switch
        {
            Student student => student switch
            {
                _ when student.Year < 10 and student.Year > 6 => "Student in Key Stage 3",
                _ when student.Year >= 10 and student.Year <= 13> => "Student in Key Stage 4",
                _ => $"Student in Year {student.Year}"
            },
            Teacher teacher => $"Teacher of {teacher.Subject}",
            _ => "Person is not a student or a teacher"
        };
        
        return result;
    }
}
```

## Tuple Patterns (with Switch Expressions)

You can also use Switch Expressions with Tuples to write even more useful code. For example, you could be creating a game with crafting, combining two items to make another.

```csharp linenums="1"
public static class YourClassHere
{
    public static CraftingMaterial GetCraftingMaterial (CraftingMaterial item1, CraftingMaterial item2)
    {
        return (item1, item2) switch
        {
            // Match the items in both positions
            (CraftingMaterial.MountainFlowers, CraftingMaterial.SoulGems) => CraftingMaterial.DwemerMetal,
            (CraftingMaterial.SoulGems, CraftingMaterial.MountainFlowers) => CraftingMaterial.DwemerMetal,

            (CraftingMaterial.Ore, CraftingMaterial.DwemerMetal) => CraftingMaterial.Ingots,
            (CraftingMaterial.DwemerMetal, CraftingMaterial.Ore) => CraftingMaterial.Ingots,

            // Handle both items being the same (discard for both, to match any)
            (_, _) when item1 == item2 => item1,

            // Default case (with discard)
            _ => CraftingMaterial.Unknown
        };
    }
}
```

This is lovely and easy to read, as well as very powerful.
