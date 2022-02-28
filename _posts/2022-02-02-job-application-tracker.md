---
layout: post
title:  "Terminal Tool for Tracking Job Applications"
date:   2022-02-02 09:30:00 -0800
categories: tools
published: true
lede: "Like any good software engineer, before I started my job search, I first needed to write my own software to track my progress."
---

Each time I set out to search for a new job, I find myself frustrated keeping track of possible jobs in spreadsheets. I'm typically initially interested in looking at a handful of certain companies to see if they have openings, but at some point that list becomes long and unwieldy as I apply to more and more roles. 

Like many data tracking problems I've experienced in the nonprofit world, the data itself isn't overly complicated or overwhlemingly large. The problem is more about finding the right way to interact with it. In this case, I wanted a terminal-based UI that would allow me to make a list of companies I might be interested in joining, keep track of jobs I've applied to at those companies, where I am in those application processes, and who I've talked to along the way. 

This is the result:
{% 
include caption-img.html image="job-search-tools-overview.gif" caption="A brief tour through the app with sample data"
alt = "A terminal application with 4 tabs: Todo, Companies, Jobs, and People. Each tab shows several items of sample data."
%}

Check out the [Github repo](https://github.com/colinjroberts/job-search-tools) for more details and to try it out.

This post is meant to serve as a walk through of my process and as a more detailed example than what I was able to find in the urwid docs to help others get over the initial hump of figuring out how to use urwid.


## Design
I knew from the get go (and experience building something similar for a coworker), that I would want separate tabs for each main type of data with. Most or all screens would have a side bar of items that I would select, then the info for that and related items would appear in the main body area. These are some text sketches I made in the early stages of development:

```
 ┌──────────────────┐
 │┌────────────────┐│
 │├────────────────┤│
 ││ ┌─┐ ┌─────────┐││
 ││ │ │ │         │││
 ││ └─┘ └─────────┘││
 │└────────────────┘│
 └──────────────────┘


Outer/Top-most layer:
 ┌──────────────────┐
 │                  │
 │  main_pile       │
 │                  │
 │                  │
 │                  │
 │                  │
 └──────────────────┘

Inside main_pile is:
 ┌────────────────┐
 │ tab_menu       │
 ├────────────────┤
 │                │
 │ body_container │
 │                │
 │                │
 └────────────────┘


Inner most containers that will hold lists and other sub-containers
 ┌───────────┐  ┌─────────────────────────────┐
 │  body_    │  │  body_main_window           │
 │  side_    │  │                             │
 │  bar      │  │                             │
 │           │  │                             │
 │           │  │                             │
 │           │  │                             │
 │           │  │                             │
 │           │  │                             │
 └───────────┘  └─────────────────────────────┘

```

## Implementation
Not wanting to write a terminal app from scratch (row and column length checking, overflow handling, style changes for selected objects, etc.), I looked for an existing toolkit. [This StackExchange answer](https://stackoverflow.com/questions/22394090/python-terminal-text-ui-tui-library) provided several options to look into. Python's [curses](https://docs.python.org/3.10/library/curses.html) module looks robust, but I couldn't find a handhold that would help me quickly move from excited ideation to a working product. [npyscreen](http://www.npcole.com/npyscreen/) had a similar issue and appeared to lack robust documentation. [Urwid](http://urwid.org) felt just right having robust documentation and enough examples to get me going.

After choosing a tool, it was a matter of following tutorials and getting something running, then slowly building up the UI box by box using mock data. 

At the very beginning, I needed something that would run and could be quit:
```python
import urwid

def exit_program():
    """Exit the main loop and return to the terminal"""
    raise urwid.ExitMainLoop()

def q_for_exit(key):
    """Check for key presses for q and quits"""
    if key in ('q', 'Q'):
        exit_program()

if __name__ == "__main__":
    """Main process that splits the main window into tabs and a main body
    using a Pile"""

    # Builds primary two subdivisions: tab_menu and body_container
    tab_menu = urwid.Text("This is where tabs will go.")
    body_container = urwid.Filler(urwid.Text("This is where "
    "the body will go."), "top")

    # Arrange primary items into a Pile
    main_pile = urwid.Pile([('pack', tab_menu), urwid.LineBox(body_container)])

    mainloop = urwid.MainLoop(main_pile,
                              palette=[('reversed', 'standout', '')],
                              unhandled_input=q_for_exit)

    mainloop.run()
```
[View this code as a Github Gist](https://gist.github.com/colinjroberts/a6b724f9ec4733287621ccd31c476e90)

Once that was in place I wanted to get the tabs for the different pages working to get a sense of how urwid handles selection events. This has two parts: making clickable tabs, then handling the clicks. To make the tabs, I followed some examples that showed using a function that takes list of strings to use as button names and returns an urwid element, in this case a GridFlow of Buttons.

```python
def build_tab_menu(choices):
    """Defines and builds tab_menu using provided choices
    Probably doesn't need to be this convoluted.
    """
    cells = []
    for item in choices:
        button = urwid.Button(item)
        urwid.connect_signal(button, 'click', on_tab_click, item)
        cells.append(urwid.AttrMap(button, None, focus_map='reversed'))
    return urwid.GridFlow(cells, 20, 2, 1, "left")
```

The full working version of unclickable tabs (well, clickable but throws an error) looks like this:

```python
import urwid

def build_tab_menu(choices):
    """Defines and builds tab_menu using list of choices"""
    cells = []
    for item in choices:
        button = urwid.Button(item)
        urwid.connect_signal(button, 'click', None, item)
        cells.append(urwid.AttrMap(button, None, focus_map='reversed'))
    return urwid.GridFlow(cells, 20, 2, 1, "left")

def exit_program():
    """Exit the main loop and return to the terminal"""
    raise urwid.ExitMainLoop()

def q_for_exit(key):
    """Check for key presses for q and quits"""
    if key in ('q', 'Q'):
        exit_program()

if __name__ == "__main__":
    """Main process that splits the main window into tabs and a main body
    using a Pile"""

    # Builds primary two subdivisions: tab_menu and body_container
    tab_menu = build_tab_menu(['Todo', 'Jobs', 'Companies', 'Contacts'])
    body_container = urwid.Filler(urwid.Text("This is where " 
    "the body will go."), "top")

    # Arrange primary items into a Pile
    main_pile = urwid.Pile([('pack', tab_menu), urwid.LineBox(body_container)])

    mainloop = urwid.MainLoop(main_pile,
                              palette=[('reversed', 'standout', '')],
                              unhandled_input=q_for_exit)

    mainloop.run()
```
[View this code as a Github Gist](https://gist.github.com/colinjroberts/a6825eab34a17efb3c5ae4435d93949a)

In `build_tab_menu`, the line `urwid.connect_signal(button, 'click', None, item)` takes each `button` and on `click` adds a function (`None`) that should be fun when clicked and passes to that function an argument `item`. That doesn't make a ton of sense because there isn't a function yet, so let's add one that will change the text of the body container to the name of the item that is passed to it.

To do that, we can use the fact that urwid components are often directly addressable to just change the content of the main body window. Remember that when the applicaiton starts, body_container is a Filler object with a body of a Text object. The following method updates that filler object with a new Text object. 

```python
def body_picker(button, choice):
    """Function for directly changing the text in body_container"""
    if choice == "Todo":
        body_container.body = urwid.Text("Todo", 'left', 'clip')
    elif choice == "Companies":
        body_container.body = urwid.Text("Companies", 'left', 'clip')
    elif choice == "Contacts":
        body_container.body = urwid.Text("Contacts", 'left', 'clip')
    elif choice == "Jobs":
        body_container.body = urwid.Text("Jobs", 'left', 'clip')
    else:
        body_container.body = urwid.Text("There must have been a mistake.",
         'left', 'clip')
```

Adding that method to the top of the file and updating the `build_tab_menu` function from `urwid.connect_signal(button, 'click', None, item)` to `urwid.connect_signal(button, 'click', body_picker, item)` should result in a fully functioning tab menu that updates the main body.

The next step is to further subdivide the body_container and building more and more support functions to read and display content provided. Here is an example with two major changes: (1) Each tab has the main body subdivided and labeled and (2) the app arranged as a class instead of as a bunch of loose functions.

```python
import urwid

class App():
    def __init__(self):

        """Main process that splits the main window into tabs and a main body
        using a Pile"""

        # Builds primary two subdivisions: tab_menu and body_container
        self.tab_menu = self.build_tab_menu(['Todo', 'Jobs', 'Companies', 'People'])
        self.body_container = urwid.Columns(self.get_body_container_columns())

        # Arrange primary items into a Pile
        self.main_pile = urwid.Pile(
            [('pack', self.tab_menu), self.body_container])

        self.mainloop = urwid.MainLoop(self.main_pile,
                                  palette=[('reversed', 'standout', '')],
                                  unhandled_input=self.q_for_exit)

        self.mainloop.run()

    def companies(self):
        """Returns a text list of company names, but in the future could contain
        an Urwid object like a ListBox of Selectable company names, each of which
        could have a callback function that would look up the related company
        information and display it in the main body window."""
        company_names = ["Albacore",
                         "BuyNLarge",
                         "Caltech",
                         "Dennys",
                         "Enron",
                         "Facebook",
                         "Google",
                        ]

        return urwid.Text("\n".join(company_names))

    def people(self):
        """Returns a text list of people names, but in the future could contain
        an Urwid object like a ListBox of Selectable people names, each of which
        could have a callback function that would look up the related person
        information and display it in the main body window."""

        contact_names = ["Alice Baker",
                         "Cooper Douglas",
                         "Eugene Fernando",
                         "Gretchen Hyacinth",
                         "Jeannie Kidseth",
                         "Liz Maroney",
                         "Norbert Ort",
                         "Penny Quinn",
                        ]
        return urwid.Text("\n".join(contact_names))

    def jobs(self):
        """Returns a text list of job names, but in the future could contain
        an Urwid object like a ListBox of Selectable job names, each of which
        could have a callback function that would look up the related person
        information and display it in the main body window."""
        job_names = ["Software Engineer",
                     "Engineer in Test",
                     "Program Manager",
                     "Product Designer",
                     "Engineering Manager",
                     "Junior Software Engineer",
                     "SDE I",
                     "Software Engineer - Backend, Finance",
                     "Manager; Software Engineering",
                     "Software Engineering and Product Design Specialist",
                     ]
        return urwid.Text("\n".join(job_names))

    def build_body_main_window(self, choice):
        """Builds the main window of the body depending on which tab is
        currently active. The calling method is a Filler, and this will return
        different Text objects to it
        """
        main_window_text = urwid.Text(
            f"This will be the main window in which {choice} data will appear",
            'center', 'clip')
        return urwid.LineBox(urwid.Filler(main_window_text, "top"),
                             title="Todo Details", title_align="left")

    def get_body_container_columns(self, choice="Todo"):
        """Builds default main body when app first runs"""
        column_1 = urwid.LineBox(
            urwid.Filler(urwid.Text("Side bar", 'center', 'clip'), "top"),
            title="Todo Details", title_align="left")
        column_2 = self.build_body_main_window(choice)
        return [("weight", 1, column_1), ("weight", 3, column_2)]

    def build_body_container(self, choice="Todo"):
        """Builds the body container"""
        return urwid.Columns(self.get_body_container_columns())

    def body_picker(self, button, choice):
        """Function for directly changing the content in body_container"""

        if choice == "Todo":
            side_bar = urwid.LineBox(
                urwid.Filler(urwid.Text("todo items", 'center', 'clip'), "top"),
                                     "Todo Title", "left")
            main_body = self.build_body_main_window(choice)
            list_of_widgets_to_return = [(side_bar, ("weight", 1, False)),
                                         (main_body, ("weight", 3, False))]

        elif choice == "Companies":
            side_bar = urwid.LineBox(urwid.Filler(self.companies(), "top"),
                                     title="Company Name", title_align="left")

            main_body_top = urwid.LineBox(
                urwid.Filler(urwid.Text("Open Jobs", 'center', 'clip'), "top"),
                title="Open Jobs", title_align="left")
            main_body_mid = urwid.LineBox(
                urwid.Filler(urwid.Text("Notes", 'center', 'clip'), "top"),
                title="Notes", title_align="left")
            main_body_bottom = urwid.LineBox(
                urwid.Filler(urwid.Text("People", 'center', 'clip'), "top"),
                title="People", title_align="left")
            main_body = urwid.Pile(
                [main_body_top, main_body_mid, main_body_bottom])

            list_of_widgets_to_return = [(side_bar, ("weight", 1, False)),
                                         (main_body, ("weight", 3, False))]

        elif choice == "People":
            side_bar = urwid.LineBox(urwid.Filler(self.people(), "top"),
                                     title="Person Name", title_align="left")

            main_body_top = urwid.LineBox(
                urwid.Filler(urwid.Text("Details", 'center', 'clip'), "top"),
                title="Details", title_align="left")
            main_body_bottom = urwid.LineBox(
                urwid.Filler(urwid.Text("Notes", 'center', 'clip'), "top"),
                title="Notes", title_align="left")
            main_body = urwid.Pile([main_body_top, main_body_bottom])

            list_of_widgets_to_return = [(side_bar, ("weight", 1, False)),
                                         (main_body, ("weight", 3, False))]

        elif choice == "Jobs":
            side_bar = urwid.LineBox(urwid.Filler(self.jobs(), "top"),
                                     title="Job Title", title_align="left")

            main_body_top = urwid.LineBox(urwid.Filler(urwid.Text(
                "Here will be a bunch of options about the job's status",
                'center', 'clip'), "top"),
                                          title="Status", title_align="left")
            main_body_mid = urwid.LineBox(urwid.Filler(urwid.Text(
                "Here will be a bunch of options about the job's posting details",
                'center', 'clip'), "top"),
                                          title="Posting Details",
                                          title_align="left")
            main_body_bottom = urwid.LineBox(urwid.Filler(urwid.Text(
                "Here will be a bunch of options about the job's notes",
                'center', 'clip'), "top"),
                                             title="Notes", title_align="left")
            main_body = urwid.Pile(
                [main_body_top, main_body_mid, main_body_bottom])

            list_of_widgets_to_return = [(side_bar, ("weight", 1, False)),
                                         (main_body, ("weight", 3, False))]


        else:
            list_of_widgets_to_return = [("pack", urwid.Text(
                "There must have been a mistake.", 'left', 'clip'))]

        self.body_container.contents = urwid.MonitoredFocusList(
            list_of_widgets_to_return)

    def build_tab_menu(self, choices):
        """Defines and builds tab_menu using list of choices"""
        cells = []
        for item in choices:
            button = urwid.Button(item)
            urwid.connect_signal(button, 'click', self.body_picker, item)
            cells.append(urwid.AttrMap(button, None, focus_map='reversed'))
        return urwid.GridFlow(cells, 20, 2, 1, "left")

    def exit_program(self):
        """Exit the main loop and return to the terminal"""
        raise urwid.ExitMainLoop()

    def q_for_exit(self, key):
        """Check for key presses for q and quits"""
        if key in ('q', 'Q'):
            self.exit_program()

if __name__ == "__main__":
    app = App()
```
[View this code as a Github Gist](https://gist.github.com/colinjroberts/172af25e535cb05825c5dcee54fb037d)


I hope those examples will help others get a faster start on their projects! Please check out the rest of it on the [Github repo](https://github.com/colinjroberts/job-search-tools) including how the pop-up window works, and how the simple SQLite database is implemented.