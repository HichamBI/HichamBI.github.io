---
layout: post
title:  "Angular 2 Testing with Webpack, Mocha, Chai and Sinon."
date:   2016-12-27 19:00:00
img: "posts/angular2.svg"
---

Hey !
Do you want to unit test an Angular 2 application with Mocha, on both, browser and/or Node. (without Karma or PhantomJs) ?

Before I wrote this post, 
I've googled to find an existing tutorial doing the same thing 
and I found a post on [Radzen blog](http://www.radzen.com/blog/testing-angular-webpack-mocha/) that launch tests on Node
and it work fine except when I've tried to test a component with an **ngFor** directive, the following error was thrown :


    Error in ./AppComponent class AppComponent - inline template:2:10
    caused by: Node is not defined


In this post, I want the share with you my solution to this issue and also the Webpack configuration
to launch test in browser with the mocha-loader plugin. 

I'll not past all the sources, you can find them [here](https://github.com/HichamBI/Angular2-Webpack-Mocha-Chai-Sinon).

Let start.

 
#### 0. Settings

Bad things first :

> **package.json :**

{% highlight json %}

{
  "scripts": {
    "start": "webpack-dev-server --inline --progress --port 8080",
    "test": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts",
    "test:watch": "rimraf .tmp && mocha-webpack --opts config/mocha/mocha-webpack.opts --watch",
    "test:server": "webpack-dev-server --config config/webpack.test.browser.js --inline --progress --port 8888"
  },
  "dependencies": {
    "@angular/common": "~2.4.1",
    "@angular/compiler": "~2.4.1",
    "@angular/core": "~2.4.1",
    "@angular/forms": "~2.4.1",
    "@angular/http": "~2.4.1",
    "@angular/platform-browser": "~2.4.1",
    "@angular/platform-browser-dynamic": "~2.4.1",
    "@angular/router": "~3.4.1",
    "core-js": "^2.4.1",
    "rxjs": "^5.0.2",
    "zone.js": "^0.7.4"
  },
  "devDependencies": {
    "@types/node": "^6.0.45",
    "angular2-template-loader": "^0.4.0",
    "awesome-typescript-loader": "^3.0.0-beta.17",
    "chai": "^3.5.0",
    "css-loader": "^0.23.1",
    "extract-text-webpack-plugin": "^1.0.1",
    "file-loader": "^0.8.5",
    "html-loader": "^0.4.3",
    "html-webpack-plugin": "^2.15.0",
    "jsdom": "^9.8.3",
    "mocha": "^3.1.2",
    "mocha-loader": "^1.0.0",
    "mocha-webpack": "^0.7.0",
    "null-loader": "^0.1.1",
    "rimraf": "^2.5.2",
    "sinon": "^2.0.0-pre.4",
    "style-loader": "^0.13.1",
    "to-string-loader": "^1.1.5",
    "typescript": "2.1.4",
    "webpack": "^2.2.0-rc.2",
    "webpack-dev-server": "^1.14.1",
    "webpack-merge": "^0.14.0",
    "webpack-node-externals": "^1.5.4"
  }
}  

{% endhighlight %}

We will use :

+ - **Webpack v2**.

+ - **mocha-webpack** and **jsdom** for tests on node.

+ - **mocha-loader** for tests on browser.


> **typings.json :**

{% highlight json %}

{
  "globalDependencies": {
    "chai": "registry:dt/chai#3.4.0+20160601211834",
    "es6-shim": "registry:dt/es6-shim#0.31.2+20160602141504",
    "mocha": "registry:dt/mocha#2.2.5+20161028141524",
    "sinon": "registry:dt/sinon#1.16.1+20161208163514"
  }
}

{% endhighlight %}


Then create an Angular 2 component : 

 > **app.component.ts :**
 
{% highlight javascript %}

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

The AppCmponent has a dependency to AppService, and for the testing part, we will use
the **sinon** fake server to mock http requests.

Now we are ready for testing.

#### 1. Node testing

In this part, we will see the webpack configuration to launch mocha test in Node environment. 

We use **mocha-webpack** to precompile bundles on server side, but is not enough,
Angular 2 (especially **Zone.js**) needs some part of DOM to work, this is why we use **jsdom** library. 

 > **webpack.test.common.js :**
 
{% highlight javascript %}

module.exports = {
    devtool: 'cheap-module-source-map',
    resolve: {
        extensions: ['.ts', '.js']
    },
    resolveLoader: {
        moduleExtensions: ['-loader']                                                      
    },
    module: {
        rules: [
            {
                test: /\.ts$/,
                loaders: ['awesome-typescript-loader', 'angular2-template-loader']
            },
            {
                test: /\.html$/,
                loader: 'html-loader'

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
            }
        ]
    },
    plugins: [
        new webpack.ContextReplacementPlugin(                           
            /angular(\\|\/)core(\\|\/)(esm(\\|\/)src|src)(\\|\/)linker/,
            __dirname                                                   
        )
    ],
};

{% endhighlight %}

Nothing special in this file, except the **resolveLoader** to bypass mocha-loader
syntax incompatibility with Webpack v2 : mocha-loader use loaders without the "-loader" suffix, which is now forbidden.

 > **webpack.test.node.js :**
 
{% highlight javascript %}

module.exports = webpackMerge(commonTestConfig, {
    target: 'node',

    externals: [
        nodeExternals()
    ]
});

{% endhighlight %}

We use **nodeExternals** to exclude node modules.

Next, we define mocha-webpack options in a separate file : 

 > **mocha-webpack.opts :**
 
{% highlight json %}

--webpack-config config/webpack.test.node.js
--require config/mocha/mocha-node-test-shim.js
test/*.spec.ts

{% endhighlight %}

The last thing to configure, is the DOM using **jsdom**. 

We create the following file that is already declared in **mocha-webpack.opts**:

 > **mocha-node-test-shim.js :**
 
{% highlight javascript %}

var jsdom = require('jsdom');
var document = jsdom.jsdom('<!doctype html><html><body></body></html>');
var window = document.defaultView;

global.document = document;
global.HTMLElement = window.HTMLElement;
global.XMLHttpRequest = window.XMLHttpRequest;
global.Node = window.Node;

require('core-js/es6');
require('core-js/es7/reflect');

require('zone.js/dist/zone');
require('zone.js/dist/long-stack-trace-zone');
require('zone.js/dist/proxy');
require('zone.js/dist/sync-test');
require('zone.js/dist/async-test');
require('zone.js/dist/fake-async-test');

var testing = require('@angular/core/testing');
var browser = require('@angular/platform-browser-dynamic/testing');

testing.TestBed.initTestEnvironment(browser.BrowserDynamicTestingModule, browser.platformBrowserDynamicTesting());

{% endhighlight %}

The first part in this file, is to initialize globals that **Zone.js** needs.

What was missing in [Radzen blog](http://www.radzen.com/blog/testing-angular-webpack-mocha/) solution, is the following instruction :

{% highlight javascript %}

global.Node = window.Node;

{% endhighlight %}

Without it, Angular 2 directives will not work.

After that, we require some Angular2 libraries.
 
Please note that now, **Zone.js**, provide a **mocha-patch.js**, it will be used in the next part.

Now we have all what we need to start writing testing.
 
{% highlight javascript %}

describe(`App Component Test`, () => {
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
            imports: [HttpModule],
            declarations: [AppComponent],
            providers: [AppService],
        }).compileComponents();
    });

    afterEach(() => {
        server.restore();
        getTestBed().resetTestingModule();
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
}

{% endhighlight %}

Launch tests with : 


    npm test
    
    Or
   
    mocha-webpack --opts config/mocha/mocha-webpack.opts

Tests results on the console :

<p align="center">  
    <img src="/img/node-tests-result.png" alt="Tests results on Node console" height: "300"/>
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
            template: helpers.root('test/mocha-index.html')
        })
    ]
});

{% endhighlight %}

We use an entry file **mocha-browser-test-shim.js** that will be loaded by mocha.
This entry file is similar to the one used for Node testing, but here we need 
to require **the mocha patch**.

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

var testing = require('@angular/core/testing');
var browser = require('@angular/platform-browser-dynamic/testing');

testing.TestBed.initTestEnvironment(browser.BrowserDynamicTestingModule, browser.platformBrowserDynamicTesting());

var context = require.context('./../../test', true, /\.spec\.ts/);
context.keys().forEach(context);
module.exports = context;

{% endhighlight %}

Launch tests with : 

    npm run test:server
    
    Or
   
    webpack-dev-server --config config/webpack.test.browser.js --inline --progress --port 8888

Go to [localhost:8888]() to see the results.


<p align="center">  
    <img src="/img/browser-tests-result.png" alt="Tests results on Node console" height: "300"/>
</p>


You can find code source [here](https://github.com/HichamBI/Angular2-Webpack-Mocha-Chai-Sinon).

Don't hesitate to comment if I missed something or you have an issue or better idea.

Thanks,

