# Incorrect API Approach

`strict_path::VirutalPath`, is NOT a lexical solution and is not relevant at all to our core anyfs design. `strict_path::VirtualPath`, only becomes relevant as an example for a backend that uses a real filesystem and still wishes to have containment.

You literally have access to the github repo:
https://github.com/DK26/strict-path-rs

`strict-path` is not part of our main project. It is only a tool for an optional backend implementation. We should not mention it outside the context of creating our VRootFsBackend implementation.

# Logical Gaps

How do we handle symlinks? If, we decided that there are no symlinks, but someone is using `strict_path::VirtualPath` in their backend implementation, they are still going to get symlink resolution by inheriting that behaviour from `VirutalPath`. We do simulate a filesystem but a backend with a real filesystem is going to have all these features still. Maybe we should rethink it?

- We must think about the fact that VRootFsBackend is not really a FileContainer, but presents us a backend implementation that uses a real filesystem. Should we seperate logically the two by providing two different crates? Like, a non-virtual vs virtual? so, non-virtual, cannot switch on or off features such as symlinks, hardlinks, etc. and would be treated differently in a manner? Would this effect our API surface?

# Wrong Naming

- anyfs-traits: That's suppose to be used to create backends, so whynot call it `anyfs-backend`? or, back to my old idea `anyfs-driver`? Sure, we may or may not have hardware in this story, but there is something about the name `driver` that seem to create intuation regarding hardware. I am not sure it the right call to name it that way. Maybe we'll stick to `backend`

- anyfs-container: While technically correct, I am on the edge here because `container` could imply Docker/Kubernetes kind of container, which it isn't. It is a storage unit though. A virtual storage unit (vsu? vsUnit? vStorageUnit? storageContainer? fileContainer? virtualStorageUnit?, virtualFileContainer? vfc?)

- `anyfs` is a cool name I came up with. However, it sounds bad in Hebrew. It might have a humor point to it "I am (ani) a looser (efes)". And it feels like a negative affirmation. I am not sure if we should stick to it.

# Target Vision

Notice that ideally, we'd like the backend to be its own constructor or builder. While the FilesContainer, should be a higher-level type that enables std::fs like APIs over the provided backend.

```rust
let storage = FilesContainer::new(
    VRootFsBackend::new()
        .with_root("/home/vroot") // path on real filesystem that is simulated into a root
        .build()
);

storage.create_dir("/user"); // in this example, the real path will be `/home/vroot/user`
storage.create_dir("/user/docs");
storage.write("/user/docs/file.txt", b"Hello, world!");
```

The backend it self, should implement traits that literally simulate everything that a Linux and/or windows filesystem feature would allow. We should, for example, be able to simulate symlink resolution by following the link, or hardlinks by following inodes, permissions, etc. Because the trait system is complete enough, we can simulate all of these features.

Should we delegate the responsibilty to the backend implementation, to optionally resolve symlinks, enable hardlinks, etc? Or should we provide a higher level API that does this?

We should, also, make our backend trait, "idiot proof" as much as we can, while giving the user all tools/APIs they might need to be able to optimize shall they choose to.

# Open Questions

- Does our current design, allow for a backend to compress/decompress or/and encrypt/decrypt files on the fly?

- How about hooks or call backs? Enabling, for example, file action event? Should we handle it at all or just let the user implement it? 

- Or, maybe hooks for some kind of potential verifications? I don't know. I guess the quetion is: should we allow hooks and callbacks?

- In anycase, should we utilize the fact we can re-invent things, to solve modern problems? Maybe we should just use the K.I.S.S principle or... maybe not?

- Should we add the optional marker generic like we did with `strict-path` crate, to allow distignuishing different file containers from each other and providing extra protection by having the compiler make sure we do not mix the environments in case we use more than one?

- I am a bit wondering if we should create a more restrective API for writing and reading bytes/data and copy files.. by introducing types. A data from one container, should not be allowed to mistakenly be written to another container? But, I can see how that could create limitations for other legit use-cases (maybe?). Maybe there could be a smart way to utilize the type system in a way.

- `agentfs` crate seem to have a goal very similar to ours. It is worth seeing what we can learn from their projcet. They do have different prioroties, but I do like their approach for auditable file system operations and maybe some other things. However, this comes back to the question of who's job it is to implement that logic and rather that should be us or the user.

https://github.com/tursodatabase/agentfs
https://crates.io/crates/agentfs/0.2.0

Maybe our project can became a superset of `agentfs`? It feels like these projects relate. At the very least for one use case. I wonder if I can or should be using `agentfs` for my own use cases. Say, a tenant isolation use case? Our curret scope is for file storage, simulated executions (not real ones), isolation and security. If we could achieve this goals with inhences features then this could fit or be helpful for our project. However, my initial intend was not to overcomplicate designs and make it highly easy and accessable. So long as we can keep it simple and easy to use, I am open to the idea of using `agentfs` as a dependency. Or some parts of it.

Also, I wonder how come he made such a reach in 2 months for this project: he has active contribution and thousands of stars. Did he publish it somewhere?

- About `vfs` crate. What kind of backends does it have? Could it serve our goals?

- What is `FUSE mount`? Would it be bad if we implemented it?

- What about `Posix Behaviour`?

- Is our use-case still compelling when we exaim what's already out there?

# Docs Improvements

- It must be clear in the introduction that AnyFS could be used to create a virtual filesystem over any kind of storage, not just a virtual root filesystem, sqlite files or memory (the default options). It is an open standard that we have created so anyone can develop their own backend implementation for their custom use-cases.

- It is wrongly implied that we use strict-path for security. This is only relevant for the VRootFsBackend implementation and not for the FilesContainer API. When we use SQLite for example, the isolation exists by simply having a different database file for each container. RAM is by default isolated by the OS, etc.

- The RAM option isn't just for testing. An Engineer may decide that their case may require using RAM as the fastest medium for their use-case.

- Backends DO NOT receive `&VirtualPath` !!!! Only the VRootFsBackend receives it. The FilesContainer API is agnostic of the path type. This is wrong!

- What the hell is a `ContainerBuilder`? We should simply have `FileContainer` or `FilesContainer` or maybe even `fsContainer` (the last one sounds best to me but may not be the best name regarding all our goals and previous discussions), whichever name fits best. Why the hell we need all of these seperated types? Try to follow Python's philosophy for having one main way of doing things and keep things "idiot proof".

- There are no `two kinds of feature selections`. runtime policy are not optional on the compilation level. We might decide to allow to optimize them in the future, if at all they would be relevant to the FilesContainer and not the Backend itself.

- WTF is `anyfs:VirtualPath` ?? This should NOT exist! We already covered why.

- Docs mention future export or import feature between host filesystem path. We do not even need this feature. If the user wants to read or write data, they already have that interface. We do not need to provide such an helper function to copy between file containers. The user should be doing the hard work wiring the file containers together. Unless, we utilize some kind of type-system protection, and then, we might wish to enable a workaround.

- Why our documents bother to mentions old, deprecated, previous design mentioned? There is only ONE design that we work on. We never implemented anything else. So there is no point mentiononing APIs that we sketched before.

- Why did we mention that Streaming I/O is a future feauture? Why not take it into account from the get go?

- path resolution, should be implemented differently for virtual backends and file containers. Real filesystem based backends, do not need us to resolve paths since they already use the OS path resolution.

- Who and why did we reject the:  design iteration: a graph-store `StorageBackend` with `NodeId`/`Edge`/`ChunkId` and mandatory transactions? It sounds wrong. What problem were we previously trying to solve with it?

