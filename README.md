# [C#] Base classes for Domain Driven Design

## Entity

```csharp
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : IEquatable<TId>
{
    public TId Id { get; protected set; }

    public bool Equals(Entity<TId> other)
    {
        if (other == null || GetType() != other.GetType())
            return false;

        return Id.Equals(other.Id);
    }

    public override bool Equals(object obj)
    {
        return Equals(obj as Entity<TId>);
    }

    public override int GetHashCode()
    {
        return $"{GetType().FullName}_{Id.ToString()}".GetHashCode();
    }

    public static bool operator ==(Entity<TId> first, Entity<TId> second)
    {
        if (first == null && second == null)
            return true;

        if ((first != null && second == null) || (first == null && second != null))
            return false;

        return first.Equals(second);
    }

    public static bool operator !=(Entity<TId> first, Entity<TId> second)
    {
        return !(first == second);
    }
}
```

## Aggregate Root
```csharp
public abstract class AggregateRoot<TId> : Entity<TId>, IEquatable<AggregateRoot<TId>>
    where TId : IEquatable<TId>
{
    public bool Equals(AggregateRoot<TId> other)
    {
        return base.Equals(other);
    }
}
```

## Value Object
```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object> GetEqualityComponents();

    public override bool Equals(object obj)
    {
        if (obj == null || GetType() != obj.GetType())
        {
            return false;
        }

        var other = (ValueObject)obj;

        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override int GetHashCode()
    {
        return GetEqualityComponents()
            .Select(x => x != null ? x.GetHashCode() : 0)
            .Aggregate((x, y) => x ^ y);
    }

    public static bool operator ==(ValueObject first, ValueObject second)
    {
        if (first == null && second == null)
            return true;

        if ((first != null && second == null) || (first == null && second != null))
            return false;

        return first.Equals(second);
    }

    public static bool operator !=(ValueObject first, ValueObject second)
    {
        return !(first == second);
    }
}
```

## Value Object with single value
Base class good for e.g. EmailAddress

```csharp
public abstract class ValueObject<T> : ValueObject
{
    protected ValueObject()
    {
    }

    protected ValueObject(T value)
    {
        Value = value;
    }

    public T Value { get; protected set; }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

## Enumeration
Use instead of C# eunums.

```csharp
public abstract class Enumeration<TId> : ValueObject
    where TId : IEquatable<TId>
{
    protected Enumeration(int id, string name)
    {
        Id = id;
        Name = name;
    }

    public int Id { get; }
    public string Name { get; }

    public static IEnumerable<T> GetAll<T>() where T : Enumeration<TId>
    {
        return typeof(T)
            .GetFields(BindingFlags.Public | BindingFlags.Static | BindingFlags.DeclaredOnly)
            .Select(f => f.GetValue(null))
            .Cast<T>();
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Id;
    }
}
```

## Fluent Validation Guard

```csharp
public static class Guard
{
    public static T Ensure<T>(string fieldName, T value, Func<IRuleBuilder<T,T>, IRuleBuilderOptions<T, T>> validation)
    {
        var validator = new GuardValidator<T>(fieldName, validation);
        validator.ValidateAndThrow(value);

        return value;
    }

    private class GuardValidator<T> : AbstractValidator<T>
    {
        public GuardValidator(string fieldName, Func<IRuleBuilder<T, T>, IRuleBuilderOptions<T, T>> validation)
        {
            validation(RuleFor(x => x))
                .OverridePropertyName(fieldName);
        }

        protected override void EnsureInstanceNotNull(object instanceToValidate)
        {
        }
    }
}
```

Example (import FluentValidation namespace)
```csharp
public Employee(Guid id)
{
    Id = Guard.Ensure(nameof(Id), id, x => x.NotEmpty());
}
```
