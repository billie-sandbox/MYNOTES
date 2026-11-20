found something, when editing category, marking delete the attribute should also be striked, and it should not appear agian in the dropdown.


redesign

so here is the redesign about the flow etc with the bug fix we just talk about. i think we need a unified ui for the dropdown management stuff. lets begin with the attribute.
in attribute creation of dropdown type:
- first, i think grouping can be done from the start of creation of dropdown type.
	- but i dont know what the best ux to do this. 
	- maybe if its dropdown type, there are dedicated setup page in the dialog, so instead of create attribute, its a button to setup the dropdown. 
	- in there, user can create the groups (remember all the validation, confirm to me all the validation that need to take place here in the client and also server side)
		- i think in desktop or medium screen, its 2 col, left is for the group making. field and then add button, enter press for add the group, and auto focus to the field agian after adding the group
		- created group is shown below it as a list with x button for delete and edit button to edit the name (validation also work here if renamed to existing value)
	- option creation
		- its on the second column on the right
		- creation is similar with the group making in term of field and button and behaviour
		- user fill the field
		- below it is a dropdown selection of the group the option want to be put in, default to no group. 
		- press enter will add the option
		- the list of added option
			- its like my invoice note, where there are sections of each group, and then inside them is options that are listed on that corresponding group
			- first one is no group section
			- user can drag options to other group section to assign them
		- the list itself:
			- it show the name of the option , and then the group label
			- it also have delete button
			- it also have edit button to edit inline the name and then the group, its a dropdown to choose from the group list
				- make sure, if use then do move the list when edit is open, its close the edit feature.
				- maybe make every click ourside the edit area, closes the edit.
- notes:
	- when in narrow screen, the two col setup naturally become stacked.

this design in creation, can be also be the same with in edit attribute 