---
layout: post
title:  "Angular Testing with Webpack, Mocha, Chai and Sinon."
date:   2016-12-27 19:00:00
img: "posts/angular2.svg"
---

(Last edit : January 16, 2018)

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

+ - **Angular v5.2.0.

+ - **Webpack v3.10.0.

+ - **jsdom v11.5.1. (need Node > 4)

+ - **mocha-webpack** for Node testing.

+ - **mocha-loader** for Browser testing.

Let's star !

#### 0. Settings

Bad things first :

> **package.json :**

{% highlight json %}
    {
      "scripts": {
        "start": "webpack-dev-server --inline --progress --port 8080",
        "test": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts",
        "test:watch": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts --watch",
        "test:report": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts --reporter=xunit --reporter-options output=tests-report.xml",
        "test:coverage": "rimraf .tmp && nyc --reporter=text --reporter=cobertura mocha-webpack --opts config/mocha/mocha-webpack.opts",
        "test:server": "webpack-dev-server --config config/webpack.test.browser.js --inline --progress --port 8888"
      },
      "dependencies": {
        "@angular/common": "5.2.0",
        "@angular/compiler": "5.2.0",
        "@angular/core": "5.2.0",
        "@angular/forms": "5.2.0",
        "@angular/http": "5.2.0",
        "@angular/platform-browser": "5.2.0",
        "@angular/platform-browser-dynamic": "5.2.0",
        "@angular/router": "5.2.0",
        "core-js": "2.5.3",
        "rxjs": "5.5.6",
        "zone.js": "0.8.20"
      },
      "devDependencies": {
        "@types/chai": "4.1.0",
        "@types/chai-spies": "0.0.0",
        "@types/mocha": "2.2.46",
        "@types/node": "9.3.0",
        "@types/sinon": "4.1.3",
        "angular2-template-loader": "0.6.2",
        "awesome-typescript-loader": "3.4.1",
        "chai": "4.1.2",
        "chai-spies": "1.0.0",
        "css-loader": "0.28.8",
        "extract-text-webpack-plugin": "3.0.2",
        "file-loader": "1.1.6",
        "html-loader": "0.5.4",
        "html-webpack-plugin": "2.30.1",
        "istanbul-instrumenter-loader": "3.0.0",
        "jsdom": "11.5.1",
        "mocha": "4.1.0",
        "mocha-loader": "1.1.2",
        "mocha-webpack": "1.0.1",
        "null-loader": "0.1.1",
        "nyc": "11.4.1",
        "rimraf": "2.6.2",
        "sinon": "4.1.6",
        "style-loader": "0.19.1",
        "to-string-loader": "1.1.5",
        "typescript": "2.6.2",
        "webpack": "3.10.0",
        "webpack-dev-server": "2.11.0",
        "webpack-merge": "4.1.1",
        "webpack-node-externals": "1.6.0"
      },
      "nyc": {
        "include": [
          "src/**/*.ts"
        ]
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
        title: string = 'Best Of Books';
        emptyMessage: string = 'Book List is empty !';

        constructor(private bookService: BookService) {
        }

        ngOnInit(): void {
            this.bookService.getBookList().then(books => this.bookList = books);
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

        getBookList(): Promise<Book[]> {
            let allBooksUrl = `http://localhost:8080/book/all`;
            return this.http.get(allBooksUrl)
                .toPromise()
                .then(response => response.json() as Book[])
                .catch(AppService.handleError);
        }

        private static handleError(error: any): Promise<any> {
            return Promise.reject(error.message || error);
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
    const webpack = require('webpack');

    module.exports = {

        resolve: {
            extensions: ['.ts', '.js']
        },

        resolveLoader: {
            moduleExtensions: ['-loader'] // To bypass mocha-loader incompatibility with webpack :
                                          // mocha-loader still using loaders without the "-loader" suffix,
                                          // which is forbidden with webpack v2++
        },

        module: {
            rules: [
                {
                    test: /\.html$/,
                    loader: 'raw-loader',
                    exclude: [helpers.root('src/test/mocha-index.html')]
                },
                {
                    test: /\.(png|jpe?g|gif|svg|woff|woff2|ttf|eot|ico)$/,
                    loader: 'null-loader'
                },
                {
                    test: /\.css$/,
                    exclude: helpers.root('src', 'app'),
                    loader: 'null-loader'
                },
                {
                    test: /\.css$/,
                    include: helpers.root('src', 'app'),
                    loader: 'raw-loader'
                },
                {
                    test: /\.json$/,
                    loader: 'json-loader',
                    exclude: [helpers.root('src/index.html')]
                }
            ]
        },

        plugins: [
            // Workaround for angular/angular#11580
            new webpack.ContextReplacementPlugin(
                /\@angular(\\|\/)core(\\|\/)esm5/,
                helpers.root('src'),
                {}
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
    global.HTMLElement = window.HTMLElement;
    global.XMLHttpRequest = window.XMLHttpRequest;
    global.Node = window.Node;
    global.Event = window.Event;

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

    module.exports = webpackMerge(commonTestConfig, {
        target: 'web',

        entry: {
            'test': 'mocha-loader!./config/mocha/mocha-browser-test-shim.js'
        },

        output: {
            path: helpers.root('tests'),
            publicPath: '/',
            filename: 'test.bundle.js'
        },

        plugins: [
            new HtmlWebpackPlugin({
                template: helpers.root('src/test/mocha-index.html')
            })
        ]
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

