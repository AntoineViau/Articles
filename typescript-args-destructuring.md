# Typescript arguments destructuring

With ES6 I can write something like this : 

    function getUser({email, slug}) {
        doSomething({email, slug});
    }
    getUser({slug});

I would like to do the same in Typescript, but with the Typescript features.  
First of all, Typescript does not assign `undefined` as default value.  
Therefore, I have to write : 

    function getUser({email = undefined, slug = undefined}) {
        doSomething({email, slug});
    }
    getUser({slug});

Now I would like to add types. Without destructuring, this would be something like : 

    function getUser({email}: {email?: string}, {slug}: {slug?: string}) {
        doSomething({email, slug});
    }

Notice we removed the explicit `undefined` assignation because we use `?` to tell the compiler our arguments are optional.  
But, doing this way, I cannot write 

    getUser({slug}); // Won't compile, signature does not match

I have to

    getUser(undefined, slug); // Ugly !

So finally, all I have to do is : 

    function getUser({email, slug}: {email?: string, slug?: string}) {
        doSomething({email, slug});
    }
    getUser({slug}); // Works !

    