# TrivialBehinds

A Micro Dependency Injector to make Windows Forms development in F# cheap and fun. Or C#.

## Installation

Available on Nuget: https://www.nuget.org/packages/TrivialBehinds/

## Elevator pitch

- You want to use the Visual Studio Forms Designer because hand writing the GUI creation code is messy.

- You don't want to couple the GUI code with Forms class because the Form class should be mostly autogenerated. Logic, event binding code etc. lives elsewhere.

- That elsewhere can be in another project, e.g. in an F# application that obviously needs to be compiled separately.

## Use case stories

- As an aspiring F# programmer, I have avoided doing Forms stuff because it's only properly supported in C# (with Designer)

- As a succesfull but aging C# programmer, I don't want to go down the rabbit hole of the more sophisticated, but
  boilerplate and abstraction ridden archituctures (Model-View-Presenter, MVVM, ...).

Both of these beautiful dreams can be realized with a tiny library, written expressly for this purpose.

## Hybrid F# - C# project

- Create solution with two projects: DesignerPart.csproj (a C# Forms application) and FsMain.fsproj (an F# console application).
- Draw and test the UI in DesignerPart
- In the end of Form1.Designer.cs (assuming you didn't bother to change the form class name), you will find these type declarations:

```csharp
//Form1.Designer.cs
private System.Windows.Forms.Button btnPlus;
private System.Windows.Forms.Label label1;
private System.Windows.Forms.Button btnMinus;
```

You need to copy-paste them to a new class within the DesignerPart project and make them public (I removed the full type qualifiers for readability):

```csharp
// Form1Ui.cs
public class Form1Ui
{
    public Button btnPlus;
    public Label label1;
    public Button btnMinus;
}
```

(Note that I didn't make the names PascalCase. Deal with it. Think of it as symmetry with originals or something.)

Now, TrivialBehinds steps in. In Form1.cs, after InitializeComponent call in ctor, we'll add the last bit of C# code we need:

```csharp
public Form1()
{
    InitializeComponent();
    // this is only thing you need to add to your form
    var d = TrivialBehinds.CreateBehind(this, new Form1Ui
    {
        btnMinus = btnMinus,
        btnPlus = btnPlus,
        label1 = label1
    });
    Deactivate += (o, e) => d.Dispose();
    // boilerplate ends
}
```

You'll note that VS autocomplete gives a pretty satisfying show when writing that Form1Ui initialization boilerplace.

What this code does is:

- It copies the ui control references that were verbosely created within InitializeComponent to a neat new class that can be passed around.
- it requests the instantiation of a "behind" class that corresponds to Form1Ui. This "behind" class will hook up the events and
  implements all the UI logic in F#. Note that the DesignerPart doesn't need dll or project reference to the "behind" part.

### The F# side

Huh, now to wash away that dirty feeling of authoring C# by switching to F# side:

- Create "New F# Console application" (there is no Forms option)
- Add a project reference to DesignerPart (because you will find both Form1 and Form1Ui class there)
- Implement the "Behind" part, which has all the real logic:

```fsharp
type Form1Behind(ui: Form1Ui) =
    let mutable counter = 0

    let showCount() =
        ui.label1.Text <- sprintf "Count is: %d" counter

    do
        ui.btnPlus.Click.Add <|
            fun _ ->
                counter <- counter+1
                showCount()
        ui.btnMinus.Click.Add <|
            fun _ ->
                counter <- counter-1
                showCount()
```


- Modify the 'main' like so:

```fsharp
[<EntryPoint; STAThread>]
let main argv =
    TrivialBehinds.RegisterBehind<Form1Ui, Form1Behind>()
    use form = new Form1()
    Application.Run(form)
    0
```

That's it! Set the F# side as startup project and press f5 to debug to your heart's content.

If that extra terminal window on launch bothers you, change outputtype form Exe to WinExe in FsMain.fsproj:

```xml
<OutputType>WinExe</OutputType>
```

## Extra notes

- Windows Forms apps have fuzzy font rendering on Windows 10, unless you do a [trick](https://docs.microsoft.com/en-us/dotnet/framework/winforms/high-dpi-support-in-windows-forms)
and compile with .NET Framework 4.7. Don't even think about publishing a Windows Forms
app without this.
