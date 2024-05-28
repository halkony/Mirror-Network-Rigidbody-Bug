# Reproducing collision bug mirror

This example was tested using Unity 2022.3.29f1 and Mirror 89.0.0.

# Description
Whenever a client authoritative Reliable Rigidbody 2D is present on a remote client's player object, the local client / server does not fire `OnCollisionEnter2D` in response to that object's collisions.

# Reproduction
1. Clone this project and open in Unity.
2. Open a second copy of the project using the Parrel Sync clones manager.
3. Run the server on the first copy. Note that the collision is recognized in the debug log.
4. Connect to the server using the second copy. Note that the collision is not recognized in the server's console, but it is recognized on the remote client.

# Analysis
The code causing this behavior is in the `FixedUpdate` function of `NetworkRigidbodyReliable2D.cs`

```
void FixedUpdate()
{
    // who ever has authority moves the Rigidbody with physics.
    // everyone else simply sets it to kinematic.
    // so that only the Transform component is synced.

    // host mode
    if (isServer && isClient)
    {
        // in host mode, we own it it if:
        // clientAuthority is disabled (hence server / we own it)
        // clientAuthority is enabled and we have authority over this object.
        bool owned = !clientAuthority || IsClientWithAuthority;

        // only set to kinematic if we don't own it
        // otherwise don't touch isKinematic.
        // the authority owner might use it either way.
        if (!owned) rb.isKinematic = true;
    }
    // client only
    else if (isClient)
    {
        // on the client, we own it only if clientAuthority is enabled,
        // and we have authority over this object.
        bool owned = IsClientWithAuthority;

        // only set to kinematic if we don't own it
        // otherwise don't touch isKinematic.
        // the authority owner might use it either way.
        if (!owned) rb.isKinematic = true;
    }
    // server only
    else if (isServer)
    {
        // on the server, we always own it if clientAuthority is disabled.
        bool owned = !clientAuthority;

        // only set to kinematic if we don't own it
        // otherwise don't touch isKinematic.
        // the authority owner might use it either way.
        if (!owned) rb.isKinematic = true;
    }
}
```

In this example, the first if block is what's run on the server/local client. Since the server doesn't own the object, `isKinematic` is set to true and collisions are not handled serverside. So `OnCollisionEnter2D` is not called. If you change this code to `bool owned = true;` (so the server handles the physics), you would get the expected behavior.
