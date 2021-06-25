### theia常见问题



##### Error: Module did not self-register

```
(base) krait>theia$ yarn start:electron
yarn run v1.22.10
$ yarn rebuild:electron && yarn --cwd examples/electron start
$ theia rebuild:electron
native node modules are already rebuilt for electron
$ theia start --plugins=local-dir:../../plugins
/System/Volumes/Data/work/develop/github.com/eclipse/theia/node_modules/bindings/bindings.js:121
        throw e;
        ^

Error: Module did not self-register: '/System/Volumes/Data/work/develop/github.com/eclipse/theia/node_modules/drivelist/build/Release/drivelist.node'.
    at process.func [as dlopen] (electron/js2c/asar.js:140:31)
    at Object.Module._extensions..node (internal/modules/cjs/loader.js:1034:18)
    at Object.func [as .node] (electron/js2c/asar.js:140:31)
    at Module.load (internal/modules/cjs/loader.js:815:32)
    at Module._load (internal/modules/cjs/loader.js:727:14)
    at Function.Module._load (electron/js2c/asar.js:769:28)
    at Module.require (internal/modules/cjs/loader.js:852:19)
    at require (internal/modules/cjs/helpers.js:74:18)
    at bindings (/System/Volumes/Data/work/develop/github.com/eclipse/theia/node_modules/bindings/bindings.js:112:48)
    at Object.<anonymous> (/System/Volumes/Data/work/develop/github.com/eclipse/theia/node_modules/drivelist/js/index.js:25:27)
^C

```

```
#解决方案
(base) krait>theia$ yarn rebuild:browser && yarn rebuild:electron
yarn rebuild:browser && yarn rebuild:electron
yarn run v1.22.10
$ theia rebuild:browser
Reverting @theia/node-pty
Reverting drivelist
Reverting find-git-repositories
Reverting native-keymap
Reverting nsfw
✨  Done in 1.03s.
yarn run v1.22.10
$ theia rebuild:electron
Processing @theia/node-pty
Processing nsfw
Processing native-keymap
Processing find-git-repositories
Processing drivelist
✔ Rebuild Complete
✨  Done in 27.24s.
```

