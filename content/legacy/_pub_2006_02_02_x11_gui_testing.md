{
   "description" : " Driving X11 GUIs using X11::GUITest Introduction Interfaces to GUI applications like DCOP or D-BUS allow you to interact with GUI applications in order to get at their internal states or set some arbitrary states. Sometimes GUIs don't allow for...",
   "slug" : "/pub/2006/02/02/x11_gui_testing.html",
   "authors" : [
      "george-nistorica"
   ],
   "draft" : null,
   "date" : "2006-02-02T00:00:00-08:00",
   "categories" : "graphics",
   "title" : "Test-Driving X11 GUIs",
   "image" : null,
   "tags" : [
      "automated-testing",
      "gui-automation",
      "gui-testing",
      "perl-automation",
      "perl-testing",
      "x11-testing",
      "x11-guitest",
      "user-interfaces"
   ],
   "thumbnail" : "/images/_pub_2006_02_02_x11_gui_testing/111-x11_test.gif"
}



### Driving X11 GUIs using X11::GUITest

### Introduction

Interfaces to GUI applications like [DCOP](http://developer.kde.org/documentation/library/kdeqt/kde3arch/dcop.html) or [D-BUS](http://www.freedesktop.org/Software/dbus) allow you to interact with GUI applications in order to get at their internal states or set some arbitrary states.

Sometimes GUIs don't allow for such interaction and you need to "click" them. If you're writing such an application, you need some sort of regression tests for it to make sure your widget/windows are as accessible as they should be. If this is the case, there is a Perl module to help you: [X11::GUITest](https://metacpan.org/pod/X11::GUITest).

Be aware that `X11::GUITest` allows you to drive a GUI, but you can't "read" data written in a widget, such as a button or an edit box. More on this in the Limitations section below.

To install `X11::GUITest`, run:

    $ perl -MCPAN -e 'shell'
    install X11::GUITest
    quit

### A Simple Example

I've included two example programs. One is [*tested.pl*](/media/_pub_2006_02_02_x11_gui_testing/tested.pl) and it serves as an example GUI. The other is [*tester.pl*](/media/_pub_2006_02_02_x11_gui_testing/tester.pl) that starts and drives the tested program.

You need Tk installed for the tested GUI. Tk comes as a package in most GNU/Linux distributions or other \*NIX OSes. Download both files in the same folder, run *./tester.pl*, and watch.

What are they doing and how do they work?

### Starting a GUI

First thing to do prior to driving a GUI is to start the driven program. While you can use `fork` and `exec` or any other means, `X11::GUITest` comes with a routine of its own.

Use `StartApp( $tested_application );` to start a GUI, which results in starting the desired application in an asynchronous manner.

If you want to start an application and wait for it to finish before going on, use `RunApp`.

### Finding a Window

After having the GUI started, you need to search for it among the other open windows on your desktop. For this, use `FindWindowLike()`, `WaitWindowLike()`, or `WaitWindowViewable()`, depending on what you need. Their names are pretty much self-explanatory.

Usually you need to have only one instance of the tested application started:

    @windows = FindWindowLike( $tested_app_title );
    print "* Number of $tested_app_title windows found: ", scalar @windows, "\n";

    if ( @windows == 1 ) {
         print "* Only one instance found, going on ...\n";
    } else {
        print "* The number of $tested_app_title instances is different than 1\n";
        print "exiting ...\n";
        exit;
    }

`FindWindowLike()` returns a list of windows that match the search criteria, which is a regular expression to match against the window title. In case there is more than one window that matches the criteria, either you have the same window started multiple times, or the regular expression isn't specific enough.

### Sending Keyboard Events to an Application

Having found the window, (when you know that there is only one, you can access it as the first element of `@windows`, namely `$windows[0]`), you probably want to send it some keystrokes. Use `SendKeys()` to do this.

If you are having a busy X server, or just want your testing to be easy for the human eye to watch, set the delay between the keystrokes (in milliseconds) with `SetKeySendDelay()`:

    SetKeySendDelay( $delay );

To send `Alt`+`O`, followed by a delay of `$delay` milliseconds, then `e`:

    SendKeys( '%(o)e' );

Besides sending plain text to an application, like sending the infamous "Hello World" to an editor window, you may have noticed that the previous example sent a combination of keys. Do so by using modifiers. The modifier keys are:

-   `^`, `Ctrl`
-   `%`, `Alt`
-   `+`, `Shift`

The `X11::GUITest` documentation has a complete list of special keys and their "encodings."

You may also find it useful to use `QuoteStringForSendKeys()` in the case of complicated strings.

### Sending Mouse Events to an Application

Sending keys may be not enough in some situations. Having an application that has keyboard shortcuts is nice, but not all of them support it. Sometimes you may need to send mouse events.

To get the absolute position of the appropriate window on your desktop:

    my ($x, $y, $width, $height) = GetWindowPos($edit_windows[0]);

Suppose that you want to click right in the middle of it. First, compute the position of the middle of the window:

    $x += $width  / 2;
    $y += $height / 2;

Now move the mouse:

    MoveMouseAbs( $x, $y );

Then press the right mouse button:

    PressMouseButton M_RIGHT;

Do something useful, and then release the mouse button. (Don't forget to do that when you're using `PressMouseButton`; otherwise, you may experience "strange" desktop behavior when your testing application exits.)

    ReleaseMouseButton M_RIGHT

You could replace `PressMouseButton()` and `ReleaseMouseButton()` with `ClickMouseButton()` if you don't have anything to do between pressing and releasing the mouse button.

In the example programs, there's something to do--navigating the context menu with keystrokes.

### Moving a Window

This is a neat and interesting feature: the ability to move windows. While it is useful to impress your friends with having their favorite mail program moving up and down, its utility lies in the fact that you can arrange the tested windows on the desktop so they are all visible.

    MoveWindow( $window_id, $x, $y );

### Limitations

As you may have noticed reading the example code, there is almost no way of validating the fact that you are indeed interacting with the right widget or window. The functions you can use for this are `FindWindow*` or `WaitWindow*`, which return a list of windows whose titles match an arbitrary regexp, and the functions that deal with child windows, such as `IsChild()` and `GetChildWindows()`.

While you may pass the window ID to a testing program, using external means to validate the tested application (such as indicating the coordinates on the screen), the problem is that you can't grab a widget's contents.

Also, while you might be tempted to parse the child tree of an application to get from the main window to one of its children, this doesn't work every time. Plenty of GUIs spawn other windows at the top level, and the spawned windows have as root window the topmost window (which is the desktop).

Here's an example of the problem that uses Mozilla Firefox. Before running the test, you must meet some prerequisites:

-   Back up your preferences before running the tests.
-   Go to Edit -&gt; Preferences -&gt; General -&gt; Connection Settings and set it to "Direct connection to the Internet."
-   Click OK, and then OK, and close the browser.

Now run the *[Firefox.pl](/media/_pub_2006_02_02_x11_gui_testing/Firefox.pl)* example code.

The test program assumes that when the Preferences window pops up, the General menu is selected.

Open Mozilla Firefox again, go to Preferences, select the Web Features menu, click OK, and exit the browser.

Rerun the *Firefox.pl* program, and watch it.

It has no idea which menu is selected, because every menu component belongs to the same window, having the same title.

### Writing GUIs for Testability

Having in mind the strengths and weaknesses of `X11::GUITest`, it's critical to design graphical user interfaces that are easy to test. This way, you shorten your maintenance time, as you can have a tester program that can help check that the GUI hasn't lost some of its windows in the development/maintenance process.

Of course, when displaying a license text when your GUI starts, you don't have the means to check that the contents are unchanged using `X11::GUITest`.

What you can do is to ensure that all child windows are "in place" and that a user can access them in the same way as he/she could in previous versions.

If you define ways of navigating the GUI using keyboard shortcuts so that you can reach any "leaf" window starting from the top-level window, then it's trivial for a test program to navigate the same way you do and ensure that all windows are reachable as they were in previous versions.

Consider the following code based on the tested Tk program:

    $menu{'OTHER'} = $menu_bar->cascade(
        -label   => 'Other',
        -tearoff => 0,
    );

    $menu{'OTHER'}->command(
        -label   => 'Editor',
        -command => sub {
            edit_window();
        }
    );

It defines a piece of menu from the overall menu of the application. As you may notice, there are no keyboard shortcuts that you can use to access the Editor window.

Thinking of testability, you could go to some lengths to test this piece of code to ensure that the Editor window is reachable and that it indeed pops up. You could record the application's position on the screen and then click the Other button, then move the mouse over the Editor button and click it. I'm sure you can spot some caveats here, among them:

-   You need to make sure that the application is always on the screen at some known coordinates (use `GetWindowPos()`) or maybe that the test always moves the window to the same place (use `MoveWindow()`).
-   You have to take into consideration font size changes, localization, and resolution changes so that you are sure you are clicking in the right place.

This kind of testing is fragile and error-prone. You can make things simpler and more robust: add keyboard shortcuts for each action. You gain two main benefits: you make some users (like me) happier and ease the testing process. You just need to define all the "paths" that you need to "walk" and define the child window titles so you know you've reached them.

Here's a slight adjustment to the tested application so that it provides keyboard shortcuts:

    $menu{'OTHER'} = $menu_bar->cascade(
        -label   => '~Other',
        -tearoff => 0,
    );

    $menu{'OTHER'}->command(
        -label   => '~Editor',
        -command => sub {
            edit_window();
        }
    );

    sub edit_window {
        # some initialization code here ...

        $edit_window = $main_window->Toplevel();

        # Set the title of the Editor window
        $edit_window->title("This is an edit window");

        # the rest of the code here ....

    }

This piece of code is easier to test. Navigate the application until you reach the Editor window:

    SendKeys('%(o)e');

Now you should have the Editor window spawned. Grab a list of windows having the title matching the Editor window's title:

    @edit_windows = FindWindowLike( $edit_title );

Check to see whether the Editor window is present. Also, there should be only one Editor window started:

    if ( @edit_windows == 1 ) {
        # code here
    } else {
        # we have zero or more than one Editor window, so something is not quite
        # right
    }

This kind of code is easy to extend, as you can store the application window hierarchy in some external file outside of the program source (in some sort of markup language file, or anything that suits your needs). Having this external definition of the windows' hierarchy and their properties, the tester program can read the same file the tested application uses; thus, both know the same keyboard shortcuts and window titles.

Program logic errors and/or bugs in underlying libraries used are easier to catch before you release the software.

### Conclusion

As you can see, there is no easy way to test an entire GUI application with `X11::GUITest`, but you can test the important parts. Also, for some actions you can use a mixed approach, such as initiating an event using the application interface (connecting to a remote server protected with a user/password auth scheme) and picking the results from a log file.

While the testing done in the previous paragraph is necessary, it is not sufficient. It would be great if there were someone willing to pick up the module and research whether it could be possible for `X11::GUITest` to be able to fetch data from the widgets, making it possible to "read" the contents of a window (from a text widget, for example).

This kind of testing is more complete than simply driving the GUI.

Of course, you could also use `X11::GUITest` to write a "record and playback" application. You might only need `GetMousePos()`, `IsMouseButtonPressed()`, and the other mouse functions. As I said earlier, in my opinion this kind of testing is too fragile.

The problem is that you can't validate the contents of the windows.
