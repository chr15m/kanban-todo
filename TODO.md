# Main project

## Backlog

- [ ] Automation (move to 'done' when checked)
- [ ] Minimize the number of times the user is asked for access.
      If possible ask for all the access required just once.

## TODO

- [ ] Display sub-lines of the form `key: value` immediately after item
- [ ] Make it a proper PWA with the minimal-pwa technique

## Bugs

## Refactor

- [ ] Simplify drop indicator logic
- [ ] Consolidate menu creation (card menus and burger menu)
- [ ] Extract line parsing logic - separate parsing from state management
- [ ] Reduce IndexedDB boilerplate with generic DB operation wrapper
- [ ] Make createAddForm more generic to handle both columns and cards
- [ ] Extract and consolidate indent detection functions

## Done

- [x] Extract file operations into higher-level abstraction
- [x] Consolidate modal logic into generic modal handler
- [x] Warning colored note on home and about page if browser doesn't support FileSystem Access API.
- [x] Ability to delete an item, with confirmation.
- [x] When the dots menu shows on a 'completed' item the stuff behind it shows through
- [x] Pressing + in a column and then cancel removes the button entirely
- [x] Group all indented lines after each item with the item
- [x] Link to source for self-hosting
- [x] About page in top right menu
- [x] Allow scroll if the informational screen is cut off.
- [x] Can we make the columns vertically fill the available space?
- [x] Plus button in the columns to add a new item, with text input.
- [x] Ability to create columns.
- [x] Moving cards around janks the file.
- [x] When dragging between columns we should show the insertion point and insert the card at that point.
- [x] Dropdown for setting the card as checked/unchecked.
- [x] Make it look more the makesprite/concrete.css theme.
- [x] When moving between columns don't insert an extra newline before, unless moving to an empty column.
- [x] Give each column a max width of 450px and have them stretch to fill the area up to that.
- [x] Don't put carriage returns between individual TODO list items. As far as possible try to preserve original formatting, just moving individual lines around without disturbing other lines.
- [x] Ability to create a project on the projects select page, including if TODO.md is empty.
