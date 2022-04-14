## Chapter 2: Using Objects

In [Chapter 1](./ch1-object-basics.md) we covered how to define, create and take ownership of a Sui object in Move. In this chapter we will look at how to use objects that you own in Move calls.

Sui authentication mechanisms ensure only you can use objects owned by you in Move calls. (We will cover non-owned objects in future chapters.) To use an object in Move calls, pass them as parameters to an entry function. Similar to Rust, there are a few ways to pass parameters:

### Pass objects by reference
There are two ways to pass objects by reference: read-only references (`&T`) and mutable references (`&mut T`). Read-only references allow you to read data from the object, while mutable references allow you to mutate the data in the object. Let's try to add a function that would allow us to update one of `ColorObject`'s values with another `ColorObject`'s value. This will exercise using both read-only references and mutable references.

The `ColorObject` we defined in the previous chapter looks like:
```rust
struct ColorObject has key {
    id: VersionedID,
    red: u8,
    green: u8,
    blue: u8,
}
```
Now let's add this function:
```rust
/// Copies the values of `from_object` into `into_object`.
public fun copy_into(from_object: &ColorObject, into_object, &mut ColorObject, _ctx: &mut TxContext) {
    into_object.red = from_object.red;
    into_object.green = from_object.green;
    into_object.blue = from_object.blue;
}
```
> :bulb: We added a `&mut TxContext` parameter to the function signature although it's not used. This is to allow the `copy_into` function to be called as [entry function](../move.md#entry-functions) from transactions. The parameter is named with an underscore `_` prefix to tell the compiler that it won't be used and we don't get an unused parameter warning.

In the above function signature, `from_object` can be a read-only reference because we only need to read its fields; conversely, `into_object` must be a mutable reference since we need to mutate it. In order for a transaction to make a call to the `copy_into` function, **the sender of the transaction must be the owner of both of `from_object` and `into_object`**.

> :bulb: Although `from_object` is a read-only reference in this transaction, it is still a mutable object in Sui storage--another transaction could be sent to mutate the object at the same time! To prevent this, Sui must lock any mutable object used as a transaction input, even when it's passed as a read-only reference. In addition, only an object's owner can send a transaction that locks the object.

We cannot write a unit test for the `copy_into` function just yet, as the support for retrieving multiple objects of the same type from the same account is still work in progress. This will be updated as soon as we have that.

### Pass objects by value
Objects can also be passed by value into an entry function. By doing so, the object is moved out of Sui storage (a.k.a. deleted). It is then up to the Move code to decide where this object should go.

> :books: Since every [Sui object struct type](./ch1-object-basics.md#define-sui-object) must include `VersionedID` as a field, and the [VersionedID struct](../../../../sui_programmability/framework/sources/ID.move) does not have the `drop` ability, the Sui object struct type [must not](https://github.com/diem/move/blob/main/language/documentation/book/src/abilities.md#drop) have `drop` ability either. Hence, any Sui object cannot be arbitrarily dropped and must be either consumed or unpacked.

There are two ways we can deal with a pass-by-value Sui object in Move:

#### Option 1. Delete the object
If the intention is to actually delete the object, we can unpack the object. This can be done only in the module that defined the struct type, due to Move's [privileged struct operations rules](https://github.com/diem/move/blob/main/language/documentation/book/src/structs-and-resources.md#privileged-struct-operations). Upon unpacking, if any field is also of struct type, recursive unpacking and deletion will be required.

However, the `id` field of a Sui object requires special handling. We must call the following API in the [ID](../../../../sui_programmability/framework/sources/ID.move) module to signal Sui that we intend to delete this object:
```rust
public fun delete(versioned_id: VersionedID);
```
Let's define a function in the `ColorObject` module that allows us to delete the object:
```rust
    public fun delete(object: ColorObject, _ctx: &mut TxContext) {
        let ColorObject { id, red: _, green: _, blue: _ } = object;
        ID::delete(id);
    }
```
As we can see, the object is unpacked, generating individual fields. The u8 values are primitive types and can all be dropped. However the `id` cannot be dropped and must be explicitly deleted through the `ID::delete` API. At the end of this call, the object will no longer be stored on-chain.

We can add a unit test for it, as well:
```rust
let owner = @0x1;
// Create a ColorObject and transfer it to @owner.
let scenario = &mut TestScenario::begin(&owner);
{
    let ctx = TestScenario::ctx(scenario);
    ColorObject::create(255, 0, 255, ctx);
};
// Delete the ColorObject we just created.
TestScenario::next_tx(scenario, &owner);
{
    let object = TestScenario::take_object<ColorObject>(scenario);
    let ctx = TestScenario::ctx(scenario);
    ColorObject::delete(object, ctx);
};
// Verify that the object was indeed deleted.
TestScenario::next_tx(scenario, &owner);
{
    assert!(!TestScenario::can_take_object<ColorObject>(scenario), 0);
}
```
The first part is the same as what we have seen in [Chapter 1](./ch1-object-basics.md#writing-unit-tests), which creates a new `ColorObject` and puts it in the owner's account. The second transaction is what we are testing: retrieve the object from the storage and then delete it. Since the object is deleted, there is no need (in fact, it is impossible) to return it to the storage. The last part of the test checks that the object is indeed no longer in the global storage and hence cannot be retrieved from there.

#### Option 2. Transfer the object
The owner of the object may want to transfer it to another account. To support this, the `ColorObject` module will need to define a `transfer` API:
```rust
public fun transfer(object: ColorObject, recipient: address, _ctx: &mut TxContext) {
    Transfer::transfer(object, recipient)
}
```
>:bulb: One cannot call `Transfer::transfer` directly in a transaction because it doesn't have `TxContext` as the last parameter, and hence it cannot be called as an [entry function](../../../../../src/build/move#entry-functions).

Let's add a test for transferring too. First of all, we create an object in `owner`'s account and then transfer it to a different account `recipient`:
```rust
let owner = @0x1;
// Create a ColorObject and transfer it to @owner.
let scenario = &mut TestScenario::begin(&owner);
{
    let ctx = TestScenario::ctx(scenario);
    ColorObject::create(255, 0, 255, ctx);
};
// Transfer the object to recipient.
let recipient = @0x2;
TestScenario::next_tx(scenario, &owner);
{
    let object = TestScenario::take_object<ColorObject>(scenario);
    let ctx = TestScenario::ctx(scenario);
    ColorObject::transfer(object, recipient, ctx);
};
```
Note that in the second transaction, the sender of the transaction should still be `owner`, because only the `owner` can transfer the object that it owns. After the tranfser, we can verify that `owner` no longer owns the object, while `recipient` now owns it:
```rust
// Check that owner no longer owns the object.
TestScenario::next_tx(scenario, &owner);
{
    assert!(!TestScenario::can_take_object<ColorObject>(scenario), 0);
};
// Check that recipient now owns the object.
TestScenario::next_tx(scenario, &recipient);
{
    assert!(TestScenario::can_take_object<ColorObject>(scenario), 0);
};
```

### On-chain interactions
Now it's time to try this out on-chain. Assuming you have already followed the instructions in [Chapter 1](./ch1-object-basics.md#on-chain-interactions), you should already have the package published and a new object created.
Now we can try to transfer it to another account address. First let's see what other account addresses you own:
```
$ wallet addresses
```
Since the default current address is the first address, let's pick the second address in the list as the recipient. In my case, I have `1416F3D5AF469905B0580B9AF843EC82D02EFD30`. Let's save it for convenience:
```
$ export RECIPIENT=1416F3D5AF469905B0580B9AF843EC82D02EFD30
```
Now let's transfer the object to this address:
```
$ wallet call --gas-budget 1000 --package $PACKAGE --module "ColorObject" --function "transfer" --args \"0x$OBJECT\" \"0x$RECIPIENT\"
```
Now let's see what objects the `RECIPIENT` owns:
```
$ wallet objects --address $RECIPIENT
```
We should be able to see that one of the objects in the list is the new `ColorObject`! This means the transfer was successful.

Let's also try to delete this object:
```
$ wallet call --gas-budget 1000 --package $PACKAGE --module "ColorObject" --function "delete" --args \"0x$OBJECT\"
```
Oops. It will error out and complain that the account address is unable to lock the object, which is a valid error because we have already transferred the object away from the original owner.

In order to operate on this object, we need to switch our wallet address to `$RECIPIENT`:
```
$ wallet switch --address $RECIPIENT
```
And try the deletion again:
```
$ wallet call --gas-budget 1000 --package $PACKAGE --module "ColorObject" --function "delete" --args \"0x$OBJECT\"
```
In the output, you will see in the `Transaction Effects` section a list of deleted objects.
This shows that the object was successfully deleted. If we run this again:
```
$ wallet objects --address $RECIPIENT
```
We will see that this object is no longer there in the wallet.

Now you know how to pass objects by reference and value and transfer them on-chain.