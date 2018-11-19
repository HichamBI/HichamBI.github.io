---
layout: post
title:  "Angular Testing with Webpack, Mocha, Chai and Sinon."
date:   2016-12-27 19:00:00
img: "posts/angular2.svg"
---

(Last edit : November 19, 2018)

Hey !
Do you want to unit test an Angular application with Mocha, on both, browser and/or Node. (without Karma or PhantomJs) ?

Before I wrote this post, 
I've googled to find an existing tutorial doing the same thing 
and I found a post on [Radzen blog](http://www.radzen.com/blog/testing-angular-webpack-mocha/) that explain how to launch tests on Node
and it work fine except when I've tried to test a component with an **ngFor** directive, the following error was thrown :


    Error in ./AppComponent class AppComponent - inline template:2:10
    caused by: Node is not defined


In this post, I will share with you my solution to this issue and also I will share with you the Webpack configuration
to launch test in browser with the mocha-loader plugin. 

I'll not past all the sources, you can find them [here](https://github.com/HichamBI/Angular-testing-webpack-mocha-chai-sinon).

In this tutorial we will use :

+ - **Angular v7.0.4.

+ - **Webpack v4.26.0.

+ - **jsdom v13.0.0. (need Node > 4)

+ - **mocha-webpack** for Node testing.

+ - **mocha-loader** for Browser testing.

Let's star !

#### 0. Settings

Bad things first :

> **package.json :**

{% highlight json %}
    {
      "name": "angular-webpack-mocha-chai-sinon",
      "version": "1.0.0",
      "description": "Running Angular tests with Webpack and Mocha",
      "license": "ISC",
      "repository": {
        "type": "git",
        "url": "git+https://github.com/HichamBI/Angular-testing-webpack-mocha-chai-sinon"
      },
      "keywords": [
        "angular",
        "test",
        "mocha",
        "webpack",
        "mocha-loader",
        "mocha-webpack",
        "sinon",
        "chai",
        "jsdom",
        "node"
      ],
      "author": "Hicham BOUZIDI IDRISSI",
      "bugs": {
        "url": "https://github.com/HichamBI/Angular-testing-webpack-mocha-chai-sinon/issues"
      },
      "homepage": "https://github.com/HichamBI/Angular-testing-webpack-mocha-chai-sinon#readme",
      "scripts": {
        "start": "webpack-dev-server --mode development --watch-content-base",
        "test": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts",
        "test:watch": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts --watch",
        "test:server": "webpack-dev-server --mode development --config config/webpack.test.browser.js",
        "test:coverage": "rimraf reports && cross-env NODE_ENV=coverage nyc --reporter=text --reporter=text-summary mocha-webpack --opts config/mocha/mocha-webpack.opts",
        "test:reports": "rimraf reports && cross-env NODE_ENV=coverage nyc mocha-webpack --opts config/mocha/mocha-webpack.opts --reporter=xunit --reporter-options output=reports/tests/tests-report.xml",
        "tslint": "tslint --format stylish --project tsconfig.json -c tslint.json --force"
      },
      "dependencies": {
        "@angular/common": "7.0.4",
        "@angular/compiler": "7.0.4",
        "@angular/compiler-cli": "7.0.4",
        "@angular/core": "7.0.4",
        "@angular/forms": "7.0.4",
        "@angular/http": "7.0.4",
        "@angular/language-service": "7.0.4",
        "@angular/platform-browser": "7.0.4",
        "@angular/platform-browser-dynamic": "7.0.4",
        "@angular/platform-server": "7.0.4",
        "@angular/router": "7.0.4",
        "core-js": "2.5.7",
        "rxjs": "6.3.3",
        "zone.js": "0.8.26"
      },
      "devDependencies": {
        "@types/chai": "4.1.7",
        "@types/chai-spies": "0.0.0",
        "@types/mocha": "5.2.5",
        "@types/node": "10.12.9",
        "@types/sinon": "5.0.6",
        "angular2-template-loader": "0.6.2",
        "atob": "2.1.0",
        "awesome-typescript-loader": "5.2.1",
        "chai": "4.2.0",
        "chai-spies": "1.0.0",
        "codelyzer": "4.5.0",
        "cross-env": "5.2.0",
        "css-loader": "1.0.1",
        "file-loader": "2.0.0",
        "html-loader": "0.5.5",
        "html-webpack-plugin": "3.2.0",
        "istanbul-instrumenter-loader": "3.0.1",
        "jsdom": "13.0.0",
        "lodash": "4.17.11",
        "mini-css-extract-plugin": "0.4.4",
        "mixin-deep": "2.0.0",
        "mocha": "5.2.0",
        "mocha-loader": "2.0.0",
        "mocha-webpack": "2.0.0-beta.0",
        "null-loader": "0.1.1",
        "nyc": "13.1.0",
        "rimraf": "2.6.2",
        "sinon": "7.1.1",
        "style-loader": "0.23.1",
        "to-string-loader": "1.1.5",
        "tslint": "5.11.0",
        "typescript": "3.1.6",
        "webpack": "4.26.0",
        "webpack-archive-plugin": "3.0.0",
        "webpack-cli": "3.1.2",
        "webpack-dev-middleware": "3.4.0",
        "webpack-dev-server": "3.1.10",
        "webpack-merge": "4.1.4",
        "webpack-node-externals": "1.7.2"
      },
      "nyc": {
        "include": [
          "src/**/*.ts"
        ],
        "reporter": [
          "text",
          "text-summary",
          "html"
        ],
        "instrument": false,
        "sourceMap": false,
        "report-dir": "./reports/coverage"
      }
    }
{% endhighlight %}

Then create an Angular component :

 > **app.component.ts :**
 
{% highlight typescript %}

    @Component({
      selector: 'app',
      templateUrl: './app.component.html'
    })
    export class AppComponent implements OnInit {
      bookList: Book[] = [];
      title = 'Best Of Books';
      emptyMessage = 'Book List is empty !';
    
      constructor(private bookService: AppService) {
      }
    
      ngOnInit(): void {
        this.bookService.getBookList().subscribe(books => this.bookList = books);
      }
    }

{% endhighlight %}


 > **app.component.html :**
 
{% highlight html %}

    <h2>{{ title }}</h2>
    <div>
        <span *ngIf="bookList.length === 0">{{ emptyMessage }}</span>
        <ul>
            <li *ngFor="let book of bookList">
                <span>{{book.originalTitle}}</span>
            </li>
        </ul>
    </div>
    <book-form></book-form>

{% endhighlight %} >

 > **book-form.component.ts :**

{% highlight typescript %}

    import { Component } from "@angular/core";
    import { Book } from "./book.model";

    @Component({
        selector: 'book-form',
        templateUrl: './book-form.component.html'
    })
    export class BookFormComponent {

        formTitle : String = 'Add New Book';
        submitted = false;
        model : any = new Book('', '');

        onSubmit() { this.submitted = true; }

        newBook() {
            this.model = new Book('', '');
        }
    }

{% endhighlight %}

 > **book-form.component.html :**

{% highlight html %}

    <div class="container">
        <h1>{{ formTitle }}</h1>
        <form (ngSubmit)="onSubmit()" #bookForm="ngForm">
            <div class="form-group">
                <label for="originalTitle">Original Title</label>
                <input type="text" class="form-control" id="originalTitle" [(ngModel)]="model.originalTitle"
                       name="originalTitle" required>
            </div>
            <div class="form-group">
                <label for="author">Author</label>
                <input type="text" class="form-control" id="author" [(ngModel)]="model.author" name="author">
            </div>
            <button id="submit" type="submit" class="btn btn-success" [disabled]="!bookForm.form.valid" (click)="newBook()" >Submit</button>
        </form>

        <br/>
        <span>Log : {{ model.originalTitle }}</span>
    </div>

{% endhighlight %}

 > **app.service.ts :**
 
{% highlight typescript %}

    @Injectable()
    export class AppService {

        constructor(private http: Http) {
        }

        getBookList(): Observable<Book[]> {
        const allBooksUrl = `http://localhost:8080/book/all`;
        return this.http.get<Book[]>(allBooksUrl);
        }
    }

{% endhighlight %}

Our component will display a list of book with an **ngFor** directive, 
if this list is empty, it will display an information message instead.

A Form component to add new Book

The AppCmponent has a dependency to AppService, so for the testing part, we will use
the **sinon** fake server to mock http requests.

Now we are ready for testing.

#### 1. Node testing

Here is the webpack setup to launch mocha test in Node environment.

We use **mocha-webpack** to precompile bundles on server side, but it's not enough,
Angular (especially **Zone.js**) needs some part of DOM to work, this is why we use **jsdom** library.

 > **webpack.test.common.js :**
 
{% highlight javascript %}

    const helpers = require('./helpers');
    const ContextReplacementPlugin = require('webpack/lib/ContextReplacementPlugin');
    
    process.traceDeprecation = true;
    
    module.exports = {
    
      resolve: {
        extensions: ['.ts', '.js', 'json']
      },
    
      module: {
        rules: [
          {
            test: /\.js$/,
            parser: {
              system: true // no warning : https://github.com/webpack/webpack/pull/6321
            }
          },
          {
            test: /\.html$/,
            use: 'raw-loader',
            exclude: [helpers.root('src/test/mocha-index.html')]
          },
          {
            test: /\.(png|jpe?g|gif|svg|woff|woff2|ttf|eot|ico)$/,
            use: 'null-loader'
          },
          {
            test: /\.css$/,
            exclude: helpers.root('src', 'app'),
            use: 'null-loader'
          },
          {
            test: /\.css$/,
            include: helpers.root('src', 'app'),
            use: 'raw-loader'
          },
          {
            test: /\.json$/,
            use: 'json-loader',
            exclude: [helpers.root('src/index.html')]
          }
        ]
      },
    
      plugins: [
        new ContextReplacementPlugin(
          // The (\\|\/) piece accounts for path separators in *nix and Windows
          /(.+)?angular(\\|\/)core(.+)?/,
          helpers.root('src') // location of your src
        ),
      ],
    
      performance: {
        hints: false
      }
    };

{% endhighlight %}

Nothing special in this file, except the **resolveLoader** to bypass mocha-loader
syntax incompatibility with Webpack v2 : mocha-loader use loaders without the "-loader" suffix, which is now forbidden.

 > **webpack.test.node.js :**
 
{% highlight javascript %}

    const nodeExternals = require('webpack-node-externals');
    const helpers = require('./helpers');
    const webpackMerge = require('webpack-merge');
    const commonTestConfig = require('./webpack.test.common.js');
    const isCoverage = process.env.NODE_ENV === 'coverage';
    
    const coverageRules = [].concat(
      {
        enforce: 'post',
        test: /\.(js|ts)$/,
        include: helpers.root('src'),
        exclude: [
          /\.(e2e|spec)\.ts$/,
          /node_modules/
        ],
        loaders: ['istanbul-instrumenter-loader']
      },
      {
        test: /\.ts$/,
        include: helpers.root('src'),
        exclude: [/\.e2e\.ts$/],
        loaders: [{
          loader: 'awesome-typescript-loader',
          query: {
            // use inline sourcemaps for "coverage" reporter
            sourceMap: false,
            inlineSourceMap: true,
            compilerOptions: {
              // Remove TypeScript helpers to be injected
              // below by DefinePlugin
              removeComments: true
            }
          }
        }, 'angular2-template-loader']
      });
    
    const nonCoverageRules = [{
      test: /\.ts$/,
      include: helpers.root('src'),
      exclude: [/\.e2e\.ts$/],
      loaders: [{
        loader: 'awesome-typescript-loader',
        query: {
          sourceMap: true
        }
      }, 'angular2-template-loader']
    }];
    
    module.exports = webpackMerge(commonTestConfig, {
      target: 'node',
    
      devtool: isCoverage ? 'eval' : 'inline-source-map',
    
      output: {
        // use absolute paths in sourcemaps (important for debugging via IDE)
        devtoolModuleFilenameTemplate: '[absolute-resource-path]',
        devtoolFallbackModuleFilenameTemplate: '[absolute-resource-path]?[hash]'
      },
    
      module: {
        rules: isCoverage ? coverageRules : nonCoverageRules
      },
    
      externals: [
        nodeExternals()
      ]
    });

{% endhighlight %}

We use **nodeExternals** to exclude node modules.

Next, we define mocha-webpack options in a separate file : 

 > **mocha-webpack.opts :**
 
{% highlight javascript %}

    --webpack-config config/webpack.test.node.js
    --require config/mocha/mocha-node-test-shim.js
    --mode development
    src/test/*.spec.ts

{% endhighlight %}

The last thing to configure, is the DOM using **jsdom**. 

We create the following file declared in **mocha-webpack.opts**:

 > **mocha-node-test-shim.js :**
 
{% highlight javascript %}

    require('core-js/es6');
    require('core-js/es7/reflect');
    
    require('zone.js/dist/zone-node');
    require('zone.js/dist/long-stack-trace-zone');
    require('zone.js/dist/proxy');
    require('zone.js/dist/sync-test');
    require('zone.js/dist/async-test');
    require('zone.js/dist/fake-async-test');
    
    const testing = require('@angular/core/testing');
    const browser = require('@angular/platform-browser-dynamic/testing');
    
    testing.TestBed.initTestEnvironment(browser.BrowserDynamicTestingModule, browser.platformBrowserDynamicTesting());
    
    const jsdom=  require("jsdom");
    const { JSDOM } = jsdom;
    
    const window = (new JSDOM('<!doctype html><html><body></body></html>')).window;
    const document = window.document;
    
    global.window = window;
    global.document = document;
    global.Document = document;
    global.HTMLElement = window.HTMLElement;
    global.XMLHttpRequest = window.XMLHttpRequest;
    global.Node = window.Node;
    global.Event = window.Event;
    global.Element = window.Element;
    global.navigator = window.navigator;
    global.KeyboardEvent = window.KeyboardEvent;
    
    global.localStorage = {
      store : {},
    
      getItem: function (key) {
        return this.store[key] || null;
      },
      setItem: function (key, value) {
        this.store[key] = value;
      },
      clear: function () {
        this.store = {};
      }
    };
    
    global.sessionStorage = {
      store : {},
    
      getItem: function (key) {
        return this.store[key] || null;
      },
      setItem: function (key, value) {
        this.store[key] = value;
      },
      clear: function () {
        this.store = {};
      }
    };
    
    // https://github.com/angular/material2/issues/7101
    Object.defineProperty(document.body.style, 'transform', {value: () => ({enumerable: true, configurable: true})});

{% endhighlight %}

The first part in this file, is to initialize globals that **Zone.js** needs.

What was missing in [Radzen blog](http://www.radzen.com/blog/testing-angular-webpack-mocha/) solution, is the following instruction :

{% highlight javascript %}

    global.Node = window.Node;

{% endhighlight %}

Without it, Angular directives will not work.

After that, we require some Angular libraries.
 
Please note that now, **Zone.js**, provide a **mocha-patch.js**, it will be used in the next part.

> **app.component.spec.ts :**

{% highlight typescript %}

    describe(`App Component`, () => {
        let comp: AppComponent;
        let fixture: ComponentFixture<AppComponent>;
        let server : any;

        beforeEach(() => {
            server = sinon.fakeServer.create();
            server.autoRespond = true;
            server.respondWith("GET",
                "http://localhost:8080/book/all",
                [202,
                    {"Content-Type": "application/json"},
                    '[' +
                    '{"originalTitle" :"The Hunger Games", "author" : "Suzanne Collins"},' +
                    '{"originalTitle" :"Pride and Prejudice", "author" : "Jane Austen"},' +
                    '{"originalTitle" :"The Chronicles of Narnia", "author" : "C.S. Lewis"}' +
                    ']'
                ]
            );

            TestBed.configureTestingModule({
                imports: [HttpModule, FormsModule],
                declarations: [AppComponent, BookFormComponent],
                providers: [AppService],
            }).compileComponents();
        });

        afterEach(() => {
            server.restore();
            getTestBed().resetTestingModule();
        });

        it('should display a title', () => {
            let title : any;
            fixture = TestBed.createComponent(AppComponent);

            title = fixture.debugElement.query(By.css('h2'));
            expect(title.nativeElement.textContent).to.equal('');

            fixture.detectChanges();

            title = fixture.debugElement.query(By.css('h2'));
            expect(title.nativeElement.textContent).to.equal('Best Of Books');
        });

        it('should have a non empty book List', (done) => {
            fixture = TestBed.createComponent(AppComponent);
            comp = fixture.componentInstance;

            fixture.detectChanges();
            expect(comp.bookList.length).equal(0);

            fixture.whenStable().then(() => {
                fixture.detectChanges();

                expect(comp.bookList.length).equal(3);

                let bookTitle = fixture.debugElement.query(By.css('li span'));
                expect(bookTitle.nativeElement.textContent).to.equal('The Hunger Games');

                done();
            });
        });

        it('should display a empty message when empty book list', (done) => {
            fixture = TestBed.createComponent(AppComponent);
            comp = fixture.componentInstance;

            fixture.detectChanges();
            fixture.whenStable().then(() => {
                comp.bookList = []; // Empty Book List

                fixture.detectChanges();
                let bookTitle = fixture.debugElement.query(By.css('h4'));
                expect(bookTitle).to.be.null;

                let errorMessage = fixture.debugElement.query(By.css('span'));
                expect(errorMessage.nativeElement.textContent).to.equal('Book List is empty !');
                done();
            });
        });
    });

{% endhighlight %}

> **book-form.component.spec.ts :**

{% highlight typescript %}

    let chai = require('chai') , spies = require('chai-spies');
    chai.use(spies);

    function newEvent(eventName: string, bubbles = false, cancelable = false) {
        let evt = document.createEvent('CustomEvent');  // MUST be 'CustomEvent'
        evt.initCustomEvent(eventName, bubbles, cancelable, null);
        return evt;
    }

    describe(`Book Form Component`, () => {
        let comp: BookFormComponent;
        let fixture: ComponentFixture<BookFormComponent>;

        beforeEach(() => {
            TestBed.configureTestingModule({
                imports: [FormsModule],
                declarations: [BookFormComponent],
            }).compileComponents();
        });

        afterEach(() => {
            getTestBed().resetTestingModule();
        });

        it('should display a title', () => {
            fixture = TestBed.createComponent(BookFormComponent);
            let title: any;

            title = fixture.debugElement.query(By.css('h1'));
            expect(title.nativeElement.textContent).to.equal('');

            fixture.detectChanges();

            title = fixture.debugElement.query(By.css('h1'));
            expect(title.nativeElement.textContent).to.equal('Add New Book');
        });

        it('should log book title', (done) => {
            fixture = TestBed.createComponent(BookFormComponent);
            comp = fixture.componentInstance;

            let originalTitleInput = fixture.debugElement.query(By.css('#originalTitle')).nativeElement;
            let logSpan = fixture.debugElement.query(By.css('span'));

            fixture.detectChanges();
            expect(logSpan.nativeElement.textContent).to.equal('Log : ');

            fixture.whenStable().then(() => {
                originalTitleInput.value = 'Harry Potter';
                originalTitleInput.dispatchEvent(newEvent('input'));
                fixture.detectChanges();

                expect(comp.model.originalTitle).to.equal('Harry Potter');
                expect(logSpan.nativeElement.textContent).to.equal('Log : Harry Potter');

                done();
            });
        });

        it('should call newBook function when submit button clicked', () => {
            fixture = TestBed.createComponent(BookFormComponent);
            comp = fixture.componentInstance;

            let submitButton = fixture.debugElement.query(By.css('#submit'));
            let newBookFunction = chai.spy.on(comp, 'newBook');
            let onSubmitFunction = chai.spy.on(comp, 'onSubmit');

            submitButton.triggerEventHandler('click', {});

            expect(newBookFunction).to.have.been.called();
            expect(onSubmitFunction).to.not.have.been.called();

        });

        it('should call onSubmit function when form submitted', () => {
            fixture = TestBed.createComponent(BookFormComponent);
            comp = fixture.componentInstance;

            let form = fixture.debugElement.query(By.css('form'));
            let onSubmitFunction = chai.spy.on(comp, 'onSubmit');
            let newBookFunction = chai.spy.on(comp, 'newBook');


            form.triggerEventHandler('submit', {});

            expect(newBookFunction).to.not.have.been.called();
            expect(onSubmitFunction).to.have.been.called();
        });
    });

{% endhighlight %}

Launch tests with : 


    npm test
    
    Or
   
    mocha-webpack --opts config/mocha/mocha-webpack.opts

Tests results on the console :

<p align="center">  
    <img src="/img/node-tests-result.png" alt="Tests results on Node console"/>
</p>

#### 2. Browser testing

This time, we will use mocha-loader plugin for testing :

> **webpack.test.browser.js :**
 
{% highlight javascript %}

    const helpers = require('./helpers');
    const webpackMerge = require('webpack-merge');
    const commonTestConfig = require('./webpack.test.common.js');
    const HtmlWebpackPlugin = require('html-webpack-plugin');
    
    module.exports = webpackMerge(commonTestConfig, {
      devtool: 'cheap-module-inline-source-map',
    
      target: 'web',
    
      entry: {
        'test': 'mocha-loader!./config/mocha/mocha-browser-test-shim.js'
      },
    
      output: {
        path: helpers.root('tests'),
        publicPath: '/',
        filename: 'test.bundle.js'
      },
    
      module: {
        rules: [
          {
            test: /\.ts$/,
            include: helpers.root('src'),
            exclude: [/\.e2e\.ts$/],
            use: ['awesome-typescript-loader', 'angular2-template-loader']
          }
        ]
      },
    
      plugins: [
        new HtmlWebpackPlugin({
          template: helpers.root('src/test/mocha-index.html')
        })
      ],
    
      devServer: {
        open: 'chrome',
        port: 8888,
        inline: true,
        quiet: false,
        noInfo: false,
        stats: {colors: true}
      }
    });


{% endhighlight %}

We use an entry file **mocha-browser-test-shim.js** that will be loaded by mocha.
This entry file is similar to the one used for Node testing, but here we need 
to require **the mocha patch**.

> **mocha-browser-test-shim.js :**

{% highlight javascript %}

    require('core-js/es6');
    require('core-js/es7/reflect');

    require('zone.js/dist/zone');
    require('zone.js/dist/long-stack-trace-zone');
    require('zone.js/dist/proxy');
    require('zone.js/dist/sync-test');
    require('zone.js/dist/async-test');
    require('zone.js/dist/fake-async-test');
    require('zone.js/dist/mocha-patch');

    const testing = require('@angular/core/testing');
    const browser = require('@angular/platform-browser-dynamic/testing');

    testing.TestBed.initTestEnvironment(browser.BrowserDynamicTestingModule, browser.platformBrowserDynamicTesting());

    const context = require.context('./../../src/test', true, /\.spec\.ts/);
    context.keys().forEach(context);
    module.exports = context;

{% endhighlight %}

Launch tests with : 

    npm run test:server
    
    Or
   
    webpack-dev-server --config config/webpack.test.browser.js --inline --progress --port 8888

Go to [localhost:8888]() to see the results.


<p align="center">  
    <img src="/img/browser-tests-result.png" alt="Tests results on Node console"/>
</p>


You can find sources [here](https://github.com/HichamBI/Angular-testing-webpack-mocha-chai-sinon).

Don't hesitate to comment if I missed something or you have an issue or better idea.

Hope that can help.

Thanks,

