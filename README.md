# ModelKit.Validation

A rule-based model validation utility with the power of Lambda expression


## Features

- Multiple rules for a signle model
- Freedom to express acceptance validation as lambda expression
- Attached error(s) for non-valid rule(s)
- Ability to define pre-condition(s) to a certain rule that relies on other rule(s) for expression evaluation
- Integrated seamlessly with default dependency injection container

## Basic Demo

1. Create a your own model. In this example, lets create a Person class as follow:

```cs
public class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }
    public string Country { get; set; }
}
```

2. Define a set of rules* for a model (ex. Person) by implementing ```IValidationRulesDefinition<T>``` with ```Rule<T>```:

```cs
public class PersonRulesDefinition : IValidationRulesDefinition<Person>
{
    public IEnumerable<Rule<Person>> GetRules () => new[]
    {
        new Rule<Person>("r-1")
        {
            Accept = x => !string.IsNullOrWhiteSpace(x.FirstName),
            Error = "FirstName must not be empty",
        },
        new Rule<Person>
        {
            Accept = x => x.FirstName.Contains('a'),
            Error = "FirstName must contains 'a' character",
            PreConditions = new[] { "r-1" }
        },
        new Rule<Person>
        {
            Accept = x => x.Country == "UAE" && x.Age > 18,
            Error = "Country & Age rule violated"
        }
    };
}
```

> **Note**
> You can define rule's identifier (key) through rule's constructor or through Key property. Notice here if rule (r-1) has been failed then the next rule that relies on it will not evaluated. For additional insight about Rule, see structure section

> **Warning**
> Make sure you have defined a rule with pre-conditions after all those pre-conditions been executed first; otherwise, this rule (with pre-conditions) would be ignored by default

3. To hookup your model(s) with the rules definition, you would need to define a validator as follow:

```cs
var validator = new ModelValidator<Person, PersonRulesDefinition>();
```
4. Call evaluate method thorugh validator instance against your model

```cs
var result = validator.Evaluate(model);
```
5. You can check collected errors in case one or more rules had been violated by a given model instance 

```cs
var errors = validator.Errors;

foreach (var error in errors)
{
    Console.WriteLine($"Error = {error.Message}");
}
```

## Dependency Injection Demo

3. By following the same steps (1 & 2) from Basic Demo, you would need to define a validator service instead of a validator directly here:

```cs
var services = new ServiceCollection();

services.AddScoped<IModelValidatorService, ModelValidatorService>(options =>
{
    var validator = new ModelValidator<Person, PersonRulesDefinition>();

    var service = new ModelValidatorService();

    service.Add(validator);

    return service;
});
```

4. All you have to do now is to consume the validation service from any class by utilizing the same interface for this validator service.

```cs
var service = provider.GetService<IModelValidatorService>();
```

5. Since you can add multiple validator to your service (see step 3), you will be able to retrieve a dedicated validator by calling ```GetValidator<T>``` and do the evalution as you did before in Basic Demo

```cs
var validator = service.GetValidator<Person>();

var result = validator.Evaluate(model);

if (!result)
{
    var errors = validator.GetErrors();

    foreach (var error in errors)
    {
        Console.WriteLine($"Error = {error.Message}");
    }
}
```
### Another Option

You can avoid validator service if want to introduce a dedicated model validator by yourself or to avoid calling ```GetValidator<T>``` and use the validator right away after it had been injected. In this case, you will define the validator like this:

```cs
services.AddScoped<IModelValidator<Person>>(_ =>
{
    return new ModelValidator<Person, PersonRulesDefinition>();
});
```
It is possible to do it without a generic interface as well:

```cs
services.AddScoped<IModelValidator>(_ =>
{
    return new ModelValidator<Person, PersonRulesDefinition>();
});
```

> **Note**
> **However**, you would need to call ```ToGeneric<T>``` to be able to access Evaluate method once you have an instance of this validator like this:

```cs
var result = validator.ToGeneric<Person>().Evaluate(model);
```

## Structure
### Rule 
| Property  | Type | Purpose |
| ------------- | ------------- | ------------- |
| Key  | string | Holds a distinguished value for object  |
| Accept  | Expression<Func<T>> | Holds a lambda expression that need to be statisfied by model  |
| Error  | tuple | Holds a pair (property, message) value in case a model failed to satisfy Accept expression    |
| Result  | bool | Holds the evaluation result of Accept expression  |
| PreConditions  | string[] | An optional property to define relying rule(s) to avoid expression complexity & un-necessary evaluation |


## License

Distributed under the MIT License.(https://choosealicense.com/licenses/mit/)
