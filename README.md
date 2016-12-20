# general
This project is a shell to contain tickets and information not directly related to any particular codebase in the pilosa organization.

# Dev rules and guidelines
## Zenhub
Use the Zenhub board to look at the overall state of work.
### New pipeline 
This is where issues and pull requests initially go. It is reviewed about once a week, and things are pulled into other pipelines.
### Icebox pipeline
The icebox contains tickets that are not eligible to be worked on immediately, but we are still tracking for some reason. 
### Backlog pipeline
This contains groomed tickets and pull requests in rough order of priority that are ready to be worked on. Those looking for work should try to pull from near the top of the backlog.
### In Progress pipeline
This contains tickets and pull requests which are assigned and are actively being worked on.
### Review/QA
When you open a pull request for a ticket and are no longer actively working on it, move it to the review/QA pipeline. The pull should go to the backlog, and then to the "In Progress" pipeline once a reviewer picks it up.
### Closed
Stuff goes here when it's done.

## Workflow
When you begin working on a ticket, assign it to yourself, and move it to the "In Progress" pipeline. If the ticket you are working on is part of an epic, there may be a branch specifically for that epic, and you should open a pull request into that branch. Otherwise, open a pull request into master. Assign the pull request to yourself, request a reviewer(s), and move the pull request to the top of the Backlog pipeline. When a reviewer starts working on a pull request, she should assign it to herself as well (you can have multiple assignees), and move it to the "In Progress" pipeline. Once you've opened the pull request, you can move your ticket into the Review/QA pipeline if you don't intend to do any more work on it.
