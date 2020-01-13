---
layout: post
title:  "Testing content tree via mocked item axes"
date:   2019-12-30 02:02:00 +0200
tags: [unit-testing]
---

In the [previous post](/2019/12/25/switching-from-fakedb-to-mocks.html) I showed that FakeDb is not the only way of doing unit testing in Sitecore nowadays. Today I want to grab a real class with tests on FakeDb and create an alternative tests suite based on mocks and AutoFixture. You will see how to grow the test setup and will have a better chance to compare two approaches.

As a system under test (SUT) I took a [small class](https://github.com/Sitecore/Helix.Examples/blob/master/examples/helix-basic-unicorn/src/Feature/Navigation/website/Services/NavigationRootResolver.cs#L9) from the [Helix Examples](https://github.com/Sitecore/Helix.Examples#welcome-to-the-sitecore-helix-examples-repository) repo. I hope no one will blame me for that. The repo contains Sitecore code samples and this particular class looks extremely attractive from the unit testing perspective. It is quite simple, in the same time it requires some configuration. In addition, this class is covered with FakeDb tests which gives us an opportunity to compare two testing techniques.

### System Under Test: NavigationRootResolver

At the time of this writing, the NavigationRootResolver looks like the following:

```cs
public class NavigationRootResolver : INavigationRootResolver
{
    public Item GetNavigationRoot(Item contextItem)
    {
        if (contextItem == null)
        {
            return null;
        }

        return contextItem.DescendsFrom(Templates.NavigationRoot.Id)
            ? contextItem
            : contextItem.Axes.GetAncestors().LastOrDefault(x =>
                x.DescendsFrom(Templates.NavigationRoot.Id));
    }
}
```

Branches to test:

1. If `contextItem` is null, return null
2. If `contextItem` is based on NavigationRoot template, return context item
3. If not, return last ancestor based on the navigation template

Scenario 3 is a bit tricky. It requires the last ancestor of a given template. It is very easy to overlook this scenario and I'll get back to this later. Now let's write some tests.

### Tests

The first case is trivial and does not require any testing techniques. We can even use default AutoFixture `[AutoData]` attribute since the only thing we need is an instance of a class without parameters:

```cs
[Theory, AutoData]
public void GetNavigationRootWithNullReturnsNull(
    NavigationRootResolver sut)
{
    var actual = sut.GetNavigationRoot(null);
    Assert.Null(actual);
}
```

To solve the second case we should use the `[DefaultAutoData]` attribute [created earlier](https://gist.github.com/sshushliapin/f7937ab589cc4588fc7515cbbfbc82de#file-sitecoreunittestingsamples-firstmock-L34). Just an empty item without any fields or template configuration works well here. We only need to configure the `DescendsFrom` method and it is easy to do that since our `item` is a mock:

```cs
[Theory, DefaultAutoData]
public void GetNavigationRootReturnsItemDescendantFromRootTemplate(
    NavigationRootResolver sut,
    Item expected)
{
    expected.DescendsFrom(NavigationRootResolver.NavRootTemplateId).Returns(true);
    var actual = sut.GetNavigationRoot(expected);
    Assert.Same(expected, actual);
}
```

So far, so good. We're getting closer to the most interesting part. We need to receive all the item ancestors and choose the last of them which is descendant from Navigation Root template. But first let's take a look on a FakeDb test first.

### Content tree test with FakeDb

FakeDb is all about the content. That's probably the most powerful selling point of the entire framework. You see the content, you execute your API against this content, and if it works, it will (most likely) work in a real Sitecore environment. The code below creates a navigation root template, then another template based on the root and after that three levels of the nested items.

```cs
var homeTemplate = ID.NewID;
_db = new Db
{
    new DbTemplate(Templates.NavigationRoot.Id),
    new DbTemplate(homeTemplate)
    {
        BaseIDs = new[] { Templates.NavigationRoot.Id }
    },
    new DbItem("Home", ID.NewID, homeTemplate)
    {
        new DbItem("Child")
        {
            new DbItem("Grandchild")
        }
    }
};
_rootItem = _db.GetItem("/sitecore/content/Home");
```

Now, having such a content tree, it's easy to test all the scenarios (I'll show only the last one, you can find all the rest in the [Helix Examples](https://github.com/Sitecore/Helix.Examples/blob/master/examples/helix-basic-unicorn/src/Feature/Navigation/tests/NavigationRootResolverTests.cs)):

```cs
[Fact]
public void ResolvesWhenContextItemIsGrandchild()
{
    var contextItem = _db.GetItem("/sitecore/content/Home/Child/Grandchild");
    var rootResolver = new NavigationRootResolver();
    var resolvedItem = rootResolver.GetNavigationRoot(contextItem);
    Assert.Equal(_rootItem.ID, resolvedItem.ID);
}
```

### What about mocks

With mocks there is no content 'visualized' in code. You cannot just run your API against fast in-memory database and see if it works. You have to *know* which mocked methods to configure and call. The reward is a smaller size of the Arrange block and avoiding Sitecore internals:

```cs
[Theory, DefaultAutoData]
public void GetNavigationRootReturnsLastAncestorDescendantFromRootTemplate(
    NavigationRootResolver sut,
    Item contextItem,
    Item ancestor1,
    Item expected)
{
    ancestor1.DescendsFrom(NavigationRootResolver.NavRootTemplateId).Returns(true);
    expected.DescendsFrom(NavigationRootResolver.NavRootTemplateId).Returns(true);
    contextItem.Axes.GetAncestors().Returns(new[] { ancestor1, expected });

    var actual = sut.GetNavigationRoot(contextItem);

    Assert.Same(expected, actual);
}
```

Note that nobody cares about template hierarchy here. Mocking `DescendsFrom` and `GetAncestors` methods fully covers the testing scenario.

### Extending Item Customization

Finally, we've got some good test but it's failing. The issue is in the `contextItem.Axes.GetAncestors()` call which throws Null Reference Exception. That is expected since in the `ItemCustomization` we've just created an empty item mock and have not configured any fields. Let's do that now:

```cs
item.Axes.Returns(Substitute.For<ItemAxes>(item));
```

I've also simplified a bit the item creation customization. Instead of calling NSubstitute to create `ID`, `ItemData` and `Database` I'm relying on AutoFixture now. That is the updated ItemCustomization version:

```cs
internal class ItemCustomization : ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Customize<Item>(x =>
            x.FromFactory<ID, ItemData, Database>(CreateItem)
                .OmitAutoProperties());
    }

    private static Item CreateItem(ID id, ItemData itemData, Database database)
    {
        var item = Substitute.For<Item>(id, itemData, database);
        item.Axes.Returns(Substitute.For<ItemAxes>(item));

        return item;
    }
}
```

Please note that the ItemCustomization above is not a new class. It is a new extended version of the customization created before. Typically, it grows altogether with the unit testing codebase allowing to test new and new scenarios. If we leave it aside, the created tests are [three times smaller](https://gist.github.com/sshushliapin/2469790ccec20fc21a993b3fab122d15) comparing to the [old fashion FakeDb tests](https://github.com/Sitecore/Helix.Examples/blob/master/examples/helix-basic-unicorn/src/Feature/Navigation/tests/NavigationRootResolverTests.cs).

### Missing test case

Earlier in this post I mentioned that the scenario 3 is a bit tricky. And that is why. Please take a look on this content setup:

```cs
new DbItem("Home", ID.NewID, homeTemplate)
{
    new DbItem("Child")
    {
        new DbItem("Grandchild")
    }
}
```

You see it? Only Home item is based on the Root template. This content *looks* correct. That's how one would configure it in the Content Editor. And that is why it is a trap of a Real Content. In `NavigationRootResolver` the `contextItem.Axes.GetAncestors().LastOrDefault(...);` call is never properly validated because there is only one item of this template available. Method can be replaced with the `.FirstOfDefault` and the test will still pass returning false positive result.

Of course this scenario can be easily fixed no matter if you go with FakeDb or mocks. My point is that starting from a real content and positive scenario increases chances to overlook important test cases.

### Conclusion

Both approaches are sufficient to test scenario with nested items and template hierarchy. With FakeDb you have a chance to literally *see* your content but should beware Real Content trap. With mocks you can configure only bare minimum of calls to pass test case and, as a result, have smaller tests which are easier to maintain.
