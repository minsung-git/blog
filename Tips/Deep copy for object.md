While copy data for primitive types copys data, object and array work differently. It just give it a reference pointer.

Let'see together.

I create studentB by copying studentA.
```
let studentA = {
    name = 'John'
}

let studentB = studentA
```

Then change studentA's name not studentB's name.
```
studentA.name = 'Jack'
console.log(studentB)
```

Let's see what we going to see.
```
> node example.js
{
    name: "Jack"
}
```
As this document said earler, it was caused becasue stdentB is just looking(referencing) at the data studentA. So if you assign object or array to the variable. It doesn't copy data from origin. It only has a reference. So this behavior would make an unexpected result.

So, if there is potential for copying and modifying data, you should use deep copy 1)create an object 2)then copy data using rest operator. Here is the way.
```
let studentA = {
    name = 'John'
}

let studentB = {
    ...studentA
}

studentA.name = 'Jack'
console.log(studentB)

```
Now console shows studentB's name 'John'. Now studentB is safe from any changes for studentB.
```
> node example.js
{
    name: "John"
}
```
