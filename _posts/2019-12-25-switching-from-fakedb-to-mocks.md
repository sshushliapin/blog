---
layout: post
title:  "Switching from FakeDb to mocks"
date:   2019-12-25 12:07:00 +0200
categories: 
---

Historically, Sitecore was not a unit test friendly platform. The API consisted of static managers and testing was quite cumbersome. There were different attempts to address that starting from 'unit testing' via [web runners](https://www.codeflood.net/testrunner/), using commercial tools that could 'mock' statics (like [Microsoft Fakes](https://docs.microsoft.com/en-us/visualstudio/test/isolating-code-under-test-with-microsoft-fakes?view=vs-2019)) or just relying on expenive integration testing.

### In-memory database

[FakeDb](https://github.com/sshushliapin/Sitecore.FakeDb) provided a mechanism to substitute database calls with fake in-memory data. It also provided an alternative API to set the content with minimal amount of code. It worked out, but everything has a price. First of all, till some point FakeDb has to run *real* Sitecore code. The most obvious (and annoying) indication of that is the [necessity of using license file](https://github.com/sshushliapin/Sitecore.FakeDb/wiki/Installation#applying-the-license-file) and [additional cofigiration steps](https://github.com/Sitecore/Habitat/pull/433/files).

Another potential issue is a risk of breaking and/or behavior changes in the Platform. FakeDb version 1 managed to cover wide range of Sitecore versions [7.2, 8.0, 8.1, 8.2](https://github.com/sshushliapin/Sitecore.FakeDb/blob/v1.7.4/version.props#L7-L8) with numerous updates. Version 2 works with [9.1 and 9.2](https://github.com/sshushliapin/Sitecore.FakeDb/blob/v2.0.1/version.props#L7) and version 3 had to [overcome list of breaking changes](https://github.com/sshushliapin/Sitecore.FakeDb/pull/213) to be compatible with 9.3. We cannot predict what may change in platform in future and how it can impact the tool.

FakeDb [cannot run test in parallel](https://github.com/sshushliapin/Sitecore.FakeDb/blob/master/test/Sitecore.FakeDb.Tests/App.config#L8). Probably, not the worst limitation for a tool which is able to run hundreds of tests in seconds, but anyway, just for you to know.

### An alternative

As you see, most of the issues appear because FakeDb calls Sitecore internals. Likely, starting from 8.2 Sitecore API has got abstractions which is a significant reason to reconsider existing testing approaches. No license required, no Sitecore internal calls, no parallel execution limitation. As soon as we can mock Sitecore classes, we're good to go.

It looks extremely attractive to me and that's why I'd like to start a series of posts describing how to mock different pieces of Sitecore. There are [some tools](https://github.com/smarchenko/SitecoreDI.NSubstitute.Helper) [doing that](https://github.com/dsolovay/AutoSitecore) but let's start from the beginning.

### FakeDb style of unit testing

In FakeDb, a typical unit test looks like [this](https://github.com/sshushliapin/Sitecore.FakeDb#sitecore-fakedb):

```cs
[Fact]
public void HowToCreateSimpleItem()
{
    using (var db = new Db
    {
        new DbItem("Home") { { "Title", "Welcome!" } }
    })
    {
        Sitecore.Data.Items.Item home = db.GetItem("/sitecore/content/home");
        Xunit.Assert.Equal("Welcome!", home["Title"]);
    }
}
```

The few lines of code above do the following:

- create an auto-generated template with `Title` field
- create an item based on this template
- store the item under specific location so that it can be retrieved by path
- set `Title` field value

Looks straightforward and familiar. Let's see how this test can be rewritten with mocks.

### Mocking Sitecore with AutoFixture

Same properties but configured in the different way:

```cs
[Theory, DefaultAutoData]
public void GetItem(Database db, Item item)
{
    item["Title"].Returns("Welcome!");
    db.GetItem("/sitecore/content/home").Returns(item);
    Assert.Equal("Welcome!", item["Title"]);
}
```

As you could noticed, there is no db context. You've got item and database variables and you're free to configure them the way you need. There is a couple of steps you need to perform to make this code working, but the good news are that you need to do that *once* when starting a new project. Let's set up a solution first.

## Toolset

In my posts I'll use [xUnit](https://xunit.net/), [NSubstitute](https://nsubstitute.github.io/) and [AutoFixture](https://github.com/AutoFixture/AutoFixture). [NUnit](https://nunit.org/), and [Moq](https://github.com/Moq/moq4/wiki/Quickstart) would work as well, that's just a matter of preference. But AutoFixture (AF) - that what makes the difference. This tool can help significantly if used the right way. If you not familiar with AF, please reffer to the [Cheat Sheet](https://github.com/AutoFixture/AutoFixture/wiki/Cheat-Sheet) page.

All we need is a class library referencing the following NuGet packages:

- [AutoFixture](https://www.nuget.org/packages/AutoFixture)
- [AutoFixture.xUnit2](https://www.nuget.org/packages/AutoFixture.xUnit2)
- [NSubstitute](https://www.nuget.org/packages/NSubstitute)
- [xunit](https://www.nuget.org/packages/xunit)
- [xunit.runner.visualstudio](https://www.nuget.org/packages/xunit.runner.visualstudio) (optional)

And the following Sitecore packages:

- [Sitecore.Kernel](https://sitecore.myget.org/feed/sc-packages/package/nuget/Sitecore.Kernel)
- [Sitecore.Nexus](https://sitecore.myget.org/feed/sc-packages/package/nuget/Sitecore.Nexus)
- [Sitecore.Nexus.Licensing](https://sitecore.myget.org/feed/sc-packages/package/nuget/Sitecore.Nexus.Licensing/2.0.6) (please note that version `2.0.6` is *newer* than `9.0.180604`)

Now, when we're done with the project setup let's look on the test in details.

### Setting up AutoFixture: Database

AutoFixture is good at creating primitive and simple types but it requires some help when dealing with abstractions or circular dependencies. We'll need to teach it how to instantiate `Database` and `Item` which contain both.

If we try to create a database using native AF `[AutoData]` attribute, the test will fail:

```cs
[Theory, AutoData]
public void CreateDatabase(Database database) // fails here
{
    Assert.NotNull(database);
}

// Error:
// AutoFixture.ObjectCreationExceptionWithPath: AutoFixture was unable to create
// an instance from Sitecore.Data.Database because creation unexpectedly failed
// with exception. Please refer to the inner exception to investigate the root
// cause of the failure.
```

AutoFixture did not manage to create an instance of abstract database type but it provides a mechanism of [customiations](https://blog.ploeh.dk/2011/03/18/EncapsulatingAutoFixtureCustomizations/) to handle such scenarios. All we need is to create a class that will instantiate a database mock:

```cs
internal class DatabaseCustomization : AutoFixture.ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Customize<Database>(x =>
            x.FromFactory(() => Substitute.For<Database>())
                .OmitAutoProperties());
    }
}
```

Now we need to apply the customization to the test *fixture* so that we receive a mocked database instance as a test parameter:

```cs
internal class DefaultAutoDataAttribute : AutoDataAttribute
{
    public DefaultAutoDataAttribute()
        : base(() => new Fixture().Customize(new DatabaseCustomization()))
    {
    }
}
```

After that we need to replace AF `[AutoData]` with our new `[DefaultAutoData]` attribute and the test is now green. We've got a mocked instance of a database:

```cs
[Theory, DefaultAutoData]
public void CreateDatabase(Database database)
{
    Assert.NotNull(database); // pass
}
```

From now on, for all the tests that requres Sitecore classes, the `[DefaultAutoData]` attribute should be used.

### Setting up AutoFixture: Item

Now when we can instatiate `Database` we can move to the next step and try to create an item. Again, it fails so we need to teach AF how to do that.

```cs
[Theory, DefaultAutoData]
public void CreateItem(Item item) // fails here
{
    Assert.NotNull(item);
}

// Error:
// AutoFixture.ObjectCreationExceptionWithPath: AutoFixture was unable to create an instance from Sitecore.Data.Items.Item.
// ...
// Inner exception messages:
// System.TypeInitializationException: The type initializer for 'Sitecore.SecurityModel.License.LicenseManager' threw an exception.
```

License Manager in the stack trace? It means that we're running *internal* Sitecore code. To make rubust tests we should avoid that and mock the item. One more customization is needed.

```cs
internal class ItemCustomization : ICustomization
{
    public void Customize(IFixture fixture)
    {
        fixture.Customize<Item>(x =>
            x.FromFactory(() => CreateItem(fixture))
                .OmitAutoProperties()
        );
    }

    private static Item CreateItem(ISpecimenBuilder fixture)
    {
        var item = Substitute.For<Item>(
            fixture.Create<ID>(),
            fixture.Create<ItemData>(),
            fixture.Create<Database>());

        return item;
    }
}
```

Please note how do we use `fixture` to create database. We can do that now since we've already created customization that is able to create it.

Now, extending the `DefaulAutoData` attribute with the new customization.

```cs
internal class DefaultAutoDataAttribute : AutoDataAttribute
{
    public DefaultAutoDataAttribute()
        : base(() => new Fixture()
            .Customize(new DatabaseCustomization())
            .Customize(new ItemCustomization()))
    {
    }
}
```

And finally, the test with mocked item is green! I'll paste it once again here:

```cs
[Theory, DefaultAutoData]
public void GetItem(Database db, Item item)
{
    item["Title"].Returns("Welcome!");
    db.GetItem("/sitecore/content/home").Returns(item);
    Assert.Equal("Welcome!", item["Title"]); // pass
}
```

Very simple, very straightforward, without limitations FakeDb tests have. If that is not a replacement of FakeDb, that is, at least, a good alternative to be aware about. The full code sample can be found [here](https://gist.github.com/sshushliapin/f7937ab589cc4588fc7515cbbfbc82de). In the next posts I'll show ho to extend the test set up and configure other item properties. Stay tuned!
