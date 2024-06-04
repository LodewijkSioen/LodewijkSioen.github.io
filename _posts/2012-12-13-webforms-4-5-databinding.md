---
title: Webforms 4.5 Databinding
date: 2012-12-13
layout: post
---

Data binding in Webforms has never been easy. Either you used the 
[Datasource Controls][1] and end up with an overcrowded markup; or you bind 
your data manually in the codebehind and end up with an overcrowded codebehind
page. No matter what you chose, ‘magic strings’ will be used and you can 
forget about testability. Those hipster MVC folks have so much easier. One 
method on the controller just supplies the view with data. No lifecycle, no 
magic strings, just clean code.

[1]: https://msdn.microsoft.com/en-us/library/system.web.ui.datasourcecontrol.aspx

<!--excerpt-->

Luckily for us, Webforms is getting a small piece of the MVC world in the 4.5 release: the old [Databound Controls][2] (repeater, gridview, listview,…) are augmented with a series of properties that greatly simplify data binding</p> <p>The first one is the [SelectMethod][3] property. You can give this property the name of a method in the codebehind. This method should return the data that the control needs to be bound to.

[2]: https://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.databoundcontrol.aspx
[3]: https://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.databoundcontrol.selectmethod.aspx

The second property is [ItemType][4]. Here you can enter the TypeName of the class that you are binding to. This will give you Intellisense support while creating your ItemTemplate.

[4]: https://msdn.microsoft.com/en-us/library/system.web.ui.webcontrols.databoundcontrol.itemtype.aspx

Let’s see how this all works together:

```aspx-cs
<asp:ListView id="ListOfUsers" runat="server" ItemType="Sioen.Layerless.Web.Pages.Account.UserModel" SelectMethod="ListUsers">
    <ItemTemplate>
	    <li>
		    <a href="<%#GetRouteUrl(" UserAction", new{action="view", id=Item.Id}) %>"><%# Item.UserName %></a>
		</li>
	</ItemTemplate>
</asp:ListView>
```


And on the server-side:

```c#
public IQueryable<UserModel> ListUsers()
{
    return Db.Query().Select(u => new UserModel{Id=u.Id, UserName=u.UserName});
}
```

Visual Studio will now light up like this, where Item will be of the type you defined in ItemType:

![Screenshot of Visual Studio showing Intellisense in asp.net markup](/assets/img/blog/WebformsDatabinding.png)

The nice thing about this is of course that you will get compilation errors when a property changes. But the best thing about this is the two-way databinding support. But that is something for a next post.