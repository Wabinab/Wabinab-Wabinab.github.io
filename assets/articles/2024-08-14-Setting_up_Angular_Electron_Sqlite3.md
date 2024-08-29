# Angular Electron Sqlite3 Combo
app technology

This is intended for personal records only. 

Upon creation of angular project (v18), there are something in tsconfig.json we need to change, so it don't bother us too often (too annoyingly). Here are they: 

```json
"emitDecoratorMetadata": true,
"strictPropertyInitialization": false,
```

The first is required by either Electron or TypeORM, one can't remember. The second is to switch off the super-annoying "you haven't initialize it" thingy. Just shut up already and let it be not initialized when it doesn't need to be! 

To install Electron, just run `npm i --save-dev electron@latest`, create an `main.js` file, and include that in `package.json`. 

```json
{
  ...,
  "main": "main.js",
  "scripts": {
    ...,
    "electron": "ng build --base-href ./ && electron ."
  }
}
```

The first will link your script, the second will allow you to call `npm run electron` instead of typing out yourself all the time, from the base folder. 

Of course, there are some contents to go into `main.js` as default. 

```js
const { app, BrowserWindow, ipcMain } = require('electron');
const url = require('url');
const path = require('path');

const { DataSource } = require('typeorm');  // use later for DB. 

let win, data_source;
function createWindow() {
  win = new BrowserWindow({
    width: 800, height: 600, webPreferences: { 
      nodeIntegration: true,
      contextIsolation: false
    }
  });

  win.loadURL(
    url.format({
      pathname: path.join(__dirname, '/dist/electron-app/browser/index.html'),
      protocol: "file:",
      slashes: true
    })
  );

  // Open DevTools (optional)
  win.webContents.openDevTools();

  initialize_db();  // this will come in later. 

  win.on('closed', function() { win = null });
}

app.on('ready', createWindow);
app.on('window-all-closed', function() {
  if (process.platform !== 'darwin') app.quit();
});
app.on('activate', function() {
  if (win == null) createWindow();
});
```

Now, we need to install TypeORM. From there website, we run: 
```bash
npm install typeorm --save
npm install reflect-metadata --save
```

Then, add `import "reflect-metadata"` in `src/main.ts` (the file that comes with Angular when you create new project). 

Then, install a SQL driver of your choice. For us, we use SQLite, so that's `npm install sqlite3 --save`. As per warning, install only **one** of them, depending on which one you use. But one isn't sure if it will not crash if you run two of them at once (perhaps because you have different needs that can only easily be solved with two different types of database). One never tried it, so tell me if you do! 

As you can see, we already have a problem. The stupid thing is, if you don't use Electron-Forge, your `main.js` is a JavaScript file, and TypeORM usually use TypeScript file. Unfortunately, you cannot use `require` to import a TypeScript into a JavaScript. The redundancy we have here is to create both the TypeScript and JavaScript and maintain both of them at the same time; which one don't have a better way to solve than using Electron Forge. But one don't like how Electron Forge simplified one's Angular platform to such basic format that one had to setup a lot more stuffs and crack my brain if one forgot to set something up. If you have a better idea of how to expand the Angular to not only "minimal" (too minimal) on Electron Forge, do tell me! The pain of setting up Angular is more than maintaining JS and TS simultaneously, in one's opinion, since the latter one can always put them at the same folder and always see them together so almost always remember to change both of them at the same time. 

Though, the problem above can be solved if you create a `main.ts` and compile it to `main.js` via this command inside your `package.json`: 
```js
"electron": "tsc main.ts && ng build --base-href ./ && electron .",
```

Do note that, your `main.js` still need to keep the same, though! But if you use that, you cannot use `TypeORM` unfortunately. You'll get class decorators error! 

> Using another name other than `main.js` is not good. Example, one use the name `electron.js` and the heck, it failed utterly. The window won't even come up. The exact reason is unknown, maybe there's clash. 

We need to add back the contents of `initialize_db()`. 

```js
function initialize_db() {
  data_source = new DataSource({
    type: "sqlite",
    synchronize: true,
    logging: false,
    // Where do you want to save your database? Will create a new DB if none already exist. 
    database: "./src/assets/data/database.sqlite",
    // Include all database you need below. 
    entities: [ require('./src/assets/models/TestItem') ]
  });

  data_source.initialize()
    .then(() => { })
    .catch((err) => console.error(err));

  ipcMain.on('get-item-by', async (event, criteria, db_name) => {
    const item_repo = data_source.getRepository(db_name);
    try { event.returnValue = await item_repo.findOneBy(criteria); } 
    catch (err) { throw err; }
  })

  ipcMain.on('get-items', async (event, db_name) => {
    const item_repo = data_source.getRepository(db_name);
    try { event.returnValue = await item_repo.find(); }
    catch (err) { throw err; }
  });

  ipcMain.on('add-item', async (event, _item, db_name) => {
    try {
      const item_repo = data_source.getRepository(db_name);
      const item = await item_repo.create(_item);
      await item_repo.save(item);
      // event.returnValue = await item_repo.find();
    } catch (err) { throw err; }
  });

  ipcMain.on('update-item', async (event, _item, db_name) => {
    const item_repo = data_source.getRepository(db_name);
    try {
      await item_repo.save(_item);  // _item MUST HAVE 'id' already!!!
      // event.returnValue = await item_repo.find()
    } catch (err) { throw err; }
  })

  ipcMain.on('delete-item', async (event, _item, db_name) => {
    try {
      const item_repo = data_source.getRepository(db_name);
      const item = await item_repo.create(_item);
      await item_repo.remove(item);
      // event.returnValue = await item_repo.find()
    } catch (err) { throw err; }
  });
}
```

Of course, ours simplify to a single data source. Practically, you could have multiple data_source especially if you have one table that needs frequent backup, another don't need, and one is big and another is small, and it makes sense to separate them. Here, we just make them as simple as possible. 

For Javascript example, [visit this site](https://orkhan.gitbook.io/typeorm/docs/usage-with-javascript). The equivalent TypeScript is easily found in the [original TypeORM website](https://typeorm.io/). These are on creating models. 

The database actually could change when you change the schema/models for TypeORM! One tried it with an empty database and it change seamlessly. However, if have data, one don't know, might need to migrate? 

Though, because javascript isn't widely documented, you may need to know how to do "unique" across multiple fields: (you add these)

```js
uniques: [
        {columns: ["page_name", "var_name"]}
    ]
```

And how about many to one relationship? Something like this, taken from [here](https://stackoverflow.com/questions/69019803/typeorm-one-to-many-relation-using-entity-schema), except we add the column,

```js
columns: {
  ...,
  ticker_data_id: { type: 'int' }
},
relations: {
  tickerData: {
    type: "many-to-one",
    target: 'TickerData',
    joinColumn: { name: 'ticker_data_id'}
  }
}
```

Without adding the column, you'll have error saying the field doesn't exist. That's because, although TypeORM create the field for us, and you can confirm the field is created by using DB Browser for SQLite to check it, TypeORM can't access the field without defining it in `columns`. The rest of the `relations` is just self-explanatory. 

### Hot Reloading
One tried `electron-reload`, `electron-reloader` without success. In the end, we install `chokidar` ourselves, and reload the window manually. So, `npm i chokidar`, then update your `main.js` as such: 

```js
const chokidar = require('chokidar');

const args = process.argv.slice(1);
let devMode = args.some(val => val === '--serve');

function createWindow() {
  ...
  if (devMode) {
    chokidar.watch(path.join(__dirname, '/dist/electron-app/browser/index.html')).on('change', (event, path) => {
      win.webContents.reload();  // or just win.reload()
    });
    win.webContents.on('did-fail-load', () => {
      win.loadURL(url.format({
        pathname: path.join(__dirname, '/dist/fin-anal/browser/index.html'),
        protocol: 'file:',
        slashes: true
      }));
    })
  }
  win.loadURL(url.format({
    pathname: path.join(__dirname, '/dist/fin-anal/browser/index.html'),
    protocol: 'file:',
    slashes: true
  }));
}

```

Note that we only watch for change in `index.html` rather than the whole folder, because if you watch the whole folder, it'll repeat the reload for all your files. Even if you don't change `index.html`, it'll change when it reloads. Don't believe one? Try it yourself! You can call `console.log("changed")` inside and see how many times it prints when it reloads! 

Note that we actually call the `win.loadURL` twice. The one outside is for initial build. The one inside is because when you call `win.webContents.reload()`, it'll point to the folder `browser` instead of to the specific `index.html`. Therefore, we need to `loadURL` one more time to point it back to the correct path. 

Finally, to auto change, you should run, on a separate terminal/powershell, `ng build --watch --base-href ./`. This will `--watch` for changes in the file and rebuild them. 