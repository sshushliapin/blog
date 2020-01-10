---
layout: post
title:  "Configuring item children"
date:   2020-01-11 11:54:00 +0200
---

Probably one of the most used item operations is requesting its children. FakeDb typically configures item hierarchy in the [db context setup](https://github.com/sshushliapin/Sitecore.FakeDb/wiki/Creating-a-Hierarchy-of-Items) block (see `using` statement). Likely, for mocks that is not needed. In this post I'll show few advanced ways to extend `ItemCustomization` in order to add subitems to a mocked item.

### FakeDb style

Let's refresh in mind how does FakeDb-style hierarchy look like:

```cs
[Fact]
public void HowToCreateHierarchyOfItems()
{
  using (Sitecore.FakeDb.Db db = new Sitecore.FakeDb.Db
    {
      new Sitecore.FakeDb.DbItem("Articles")
        {
          new Sitecore.FakeDb.DbItem("Getting Started"),
          new Sitecore.FakeDb.DbItem("Troubleshooting")
        }
    })
  {
    Sitecore.Data.Items.Item articles =
      db.GetItem("/sitecore/content/Articles");

    Xunit.Assert.NotNull(articles.Children["Getting Started"]);
    Xunit.Assert.NotNull(articles.Children["Troubleshooting"]);
  }
}
```

Looks pretty but it takes a while to get to the line of code where we can actually retrieve some children. As usual, db context needs to be created. Then we have to add root item, two child items and finally retrieve the root from the database.

With mocks and AutoFixture, children can be created in a *single* line of code using `root.Children.InnerChildren` property. Let's leave the question if that's a good design to reveal [internal implementation details](https://enterprisecraftsmanship.com/posts/what-is-implementation-detail/) hanging in the air. For us it's a back door that allows to add children to an item. Just compare this with the previous sample:

```cs
[Theory, DefaultAutoData]
public void GetChildren(
    Item root,
    Item child1,
    Item child2)
{
    root.Children.InnerChildren.AddRange(new[] { child1, child2 });

    Assert.Same(child1, root.Children[child1.ID]);
    Assert.Same(child2, root.Children[child2.ID]);
}
```

That was the minimal working example. In case we need child names, the test can be slightly extended:

```cs
[Theory, DefaultAutoData]
public void GetChildrenByName(
    Item root,
    Item child1,
    Item child2)
{
    child1.Name.Returns("Getting Started");
    child2.Name.Returns("Troubleshooting");

    root.Children.InnerChildren.AddRange(new[] { child1, child2 });

    Assert.Same(child1, root.Children["Getting Started"]);
    Assert.Same(child2, root.Children["Troubleshooting"]);
}
```

### Extending Item Customization with Children property

As you could've notice, any new scenario requires some extension of the ```ItemCustomization``` class we created [before](/2019/12/30/testing-content-tree-via-mocked-item-axes.html#extending-item-customization). ATM, the `item.Children` collection is null and needs to be configured. The following line of code must be added to the `CreateItem` method:

```cs
internal class ItemCustomization : ICustomization
{
    ...

    private static Item CreateItem(ID id, ItemData itemData, Database database)
    {
        ...
        item.Children.Returns(Substitute.For<ChildList>(item, new ItemList()));
        return item;
    }
}
```

All the tests above pass. Item returns a collection of children but there is one thing worth to be mentioned.

### Mocks are not fakes

As it says, [mocks are not fakes](https://blog.pragmatists.com/test-doubles-fakes-mocks-and-stubs-1a7491dfa3da). It means that the testing approach described above requires manual configuration of *all* the api calls. There is no any Sitecore logic running behind the scene. For instance, injecting some items into `item.Children` collection does not make `item.HasChildren` to return true. It will return false unless this specific call is configured:

```cs
[Theory, DefaultAutoData]
public void GetChildrenVsOtherProps(Item parent, Item child)
{
    parent.Children.InnerChildren.AddRange(new[] { child });
    Assert.NotEmpty(parent.Children);
    Assert.True(parent.HasChildren); // fails

    parent.HasChildren.Returns(true);
    Assert.True(parent.HasChildren); // passes
}
```

Same is valid for child parent props. If you want to call `Parent` or `ParentID`, the props must be set in advance:

```cs
[Theory, DefaultAutoData]
public void ChildrenParentProps(Item parent, Item child)
{
    parent.Children.InnerChildren.AddRange(new[] { child });
    child.Parent.Returns(parent);
    child.ParentID.Returns(parent.ID);

    Assert.Same(parent, child.Parent);
    Assert.Same(parent.ID, child.ParentID);
}
```

The pure mocks described above are sufficient and can be successfully used in real projects. This approach is straightforward. Tests suppose to configure all the methods/props needed for SUT which makes clear what is going on. Disadvantage might be the necessity to configure few relative props which sometimes might look awkward.

### Mixed approach

It is possible to combine both techniques. We can use mocked items and add some fake logic to related properties. That would reduce the amount of configuration code. See how adding a child could impact `HasChildren` property. In the `ItemCustomization`, add the following line:

```cs
internal class ItemCustomization : ICustomization
{
    ...
    private static Item CreateItem(ID id, ItemData itemData, Database database)
    {
        ...
        item.HasChildren.Returns(c => item.Children.InnerChildren.Any());
        return item;
    }
}
```

This changes test behavior. The `HasChildren` property cannot (!) be configured in test anymore (TBH, that was a surprise to me so I might need to get back to this later). Instead, what you can do is to add/remove child items and observe the prop value change:

```cs
[Theory, DefaultAutoData]
public void GetChildrenWithFakeProps(Item parent, Item child)
{
    parent.Children.InnerChildren.AddRange(new[] { child });
    Assert.True(parent.HasChildren); // pass

    parent.Children.InnerChildren.Clear();
    Assert.False(parent.HasChildren); // pass
}
```

#### Configuring related props via extension methods

Further step might be to introduce some simple logic to set all the related properties at once. For instance, new `AddRange` extension method can set child's `Parent` and `ParentID` props (the code snippet below assumes `HasChildren` is already configured via mocks as shown earlier):

```cs
[Theory, DefaultAutoData]
public void AddChildrenWithFakeProps(Item parent, Item child)
{
    parent.Children.AddRange(new[] { child });
    Assert.True(parent.HasChildren);
    Assert.Same(parent, child.Parent);
    Assert.Same(parent.ID, child.ParentID);
}

internal static class ChildListExtensions
{
    public static void AddRange(this ChildList childList, IEnumerable<Item> newItems)
    {
        var parent = childList.OwnerItem;
        foreach (var item in newItems)
        {
            childList.InnerChildren.Add(item);
            item.Parent.Returns(parent);
            item.ParentID.Returns(parent.ID);
        }
    }
}
```

I'd like to highlight that as soon as the fake logic behind mocks is trivial it's ok to keep it in the test code base. If you find it too complex and does not wish to invest time into maintaining it over time, you might be interesting in the ready-to-use library like [Sitecore.NSubstituteUtils](https://github.com/smarchenko/SitecoreDI.NSubstitute.Helper) created by [@smar](https://github.com/smarchenko).

#### Configuring related props using Sitecore.NSubstituteUtils

While all the samples above can be implemented in any testing suite, there are some public libraries that can do pretty much the same things OOTB. There is an example of item structure created with [Sitecore.NSubstituteUtils](https://github.com/smarchenko/SitecoreDI.NSubstitute.Helper#creating-item-and-structures-of-items):

```cs
[Theory, DefaultAutoData]
public void ConfigureChildrenViaNSubstituteHelper(
    FakeItem parentFake,
    FakeItem child1Fake,
    FakeItem child2Fake)
{
    child1Fake.WithName("Getting Started");
    child2Fake.WithName("Troubleshooting");

    var parent = (Item)parentFake.WithChild(child1Fake).WithChild(child2Fake);
    var child1 = (Item)child1Fake;
    var child2 = (Item)child2Fake;

    Assert.Same(child1, parent.Children["Getting Started"]);
    Assert.Same(child2, parent.Children["Troubleshooting"]);
    Assert.True(parent.HasChildren);
    Assert.Same(parent, child1.Parent);
    Assert.Same(parent, child2.Parent);
    Assert.Same(parent.ID, child1.ParentID);
    Assert.Same(parent.ID, child2.ParentID);
}
```

Following this approach you won't need to introduce extension methods and implement fake item logic. Disadvantage (IMHO) is yet another alternative item management API (like in FakeDb). I'd like Sitecore to let us use default API without the necessity to jump between `FakeItem`/`DbItem` and `Item`.

### Conclusion

There is a wide list of options how to set item children in tests. You can use old good FakeDb, you can configure simple mocks per prop/method call, you can combine mocks with fake logic to impact related props or you can use existing tools like Sitecore.NSubstituteUtils. Of course, it's up to you to choose the most suitable approach.

All the test samples can be found in the [SitecoreUnitTestingSamples](https://github.com/sshushliapin/SitecoreUnitTestingSamples/tree/master/SitecoreUnitTestingSamples) GitHub repo.
