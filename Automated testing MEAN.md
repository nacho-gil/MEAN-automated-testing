# Automating Unit and Integration testing with the MEAN stack

These days, the MEAN stack - [MongoDB](http://www.mongodb.org/), [Express](http://expressjs.com), [AngularJS](http://angularjs.org), and [Node.js](http://nodejs.org) - has become a relatively mature strategy to approach a specific kind of web projects with the advantages of using Javascript as a uniform language throughout: an increase of productivity where client side developers can easily understand the server side code and database objects with little effort.

At AKQA, when we first started working on AngularJS projects, we soon realised how easily we could rely on testing our code to give developers the confidence of delivering good quality software. The answer was [Karma](http://karma-runner.github.io/), a test runner created by the AngularJS team to bring a productive testing environment to AngularJS developers.

### Karma
> On the AngularJS team, we rely on testing and we always seek better tools to make our life easier. That's why we created Karma - a test runner that fits all our needs.

Karma is **framework agnostic**. Which means that tests can be written using the framework of your choice e.g. [Jasmine](http://jasmine.github.io/) or [Mocha](http://mochajs.org/). Besides, it also supports **preprocessors** - a very handy feature when an application has an intermediate build step, for instance if you want to use CommonJS modules in your client side Javascript.

So far, this is satisfying developers that want to make sure good quality code is produced. But ultimately, we want a mechanism to **automate** running those tests as part of a **continuous integration** (CI) set-up, and extend the test execution to the **server** (Node.js). As well as, generate **Code Coverage** reports for both client and server side code so the the health of a project can be measured in real time not only by developers but by a wider audience.

Coming up with a testing strategy set-up throughout the entire stack like the described above can be time consuming. There are many articles offering recommendations but it is not easy to find advise with a unified solution, from end to end.

In this article, I'm going to cover how to set up a testing and code coverage solution for the MEAN stack using [gulp.js](http://gulpjs.com/) - *the streaming build system to automate an application workflow*.

### Automation with gulp.js
Before running any tests, build tool tasks such as [gulp-jshint](https://github.com/spalger/gulp-jshint) help in the identification of suspicious issues in your code; flagging them with a JSHint reporter like [jshint-stylish](https://github.com/sindresorhus/jshint-stylish). This task can also be run as part of the CI set-up.
```js
var gulp = require('gulp'),
    jshint = require('gulp-jshint'),
    stylish = require('jshint-stylish');

gulp.task('lint', function() {
    return gulp.src([JS_PATH + '**/*.js'])
        .pipe(jshint())
        .pipe(jshint.reporter(stylish));
});
```

When it comes to testing AngularJS projects, Karma is the right choice. Developers can choose their testing framework, for instance, I opted for **Jasmine** to test the client side Javascript.

Generating **test coverage reports** with Karma is not difficult either. First create a [Configuration file](http://karma-runner.github.io/0.12/config/configuration-file.html) and map what files do you want to generate reports from in the `preprocessors` block. Most of the preprocessors need to be loaded as plugins - other than [CoffeeScript](https://github.com/karma-runner/karma-coffee-preprocessor) or [coverage](https://github.com/karma-runner/karma-coverage) that are already available.

Nowadays, it is not unusual to use a preprocessor like [Browserify](http://browserify.org/) to write modular Javascript in the browser and then bundle up all of your dependencies. In the following example, I handle this scenario in the configuration file by loading [karma-commonjs](https://github.com/karma-runner/karma-commonjs) as a framework, so that it can be used in the `preprocessors` block later. 

You may wonder why choosing [karma-commonjs](https://github.com/karma-runner/karma-commonjs) and not [karma-browserify](https://github.com/Nikku/karma-browserify) if I am using Browserify to compile the project. The reason is simply because **coverage** and **karma-commonjs** work seamlessly if you combine them together. Something that **karma-browserify** doesn't support at the time of writing.

For example, create a **Gulpfile.js** with the following:
```js
var gulp = require('gulp'),
    browserify = require('gulp-browserify'),
    karma = require('gulp-karma'),
    uglify = require('gulp-uglify');

// Bundle up with Browserify, Minify and copy Javascript
gulp.task('scripts', function() {
    return gulp.src(JS_PATH + 'app.js')
        .pipe(browserify({
            insertGlobals: true
        }))
        .pipe(uglify())
        .pipe(gulp.dest(DIST_PATH  + 'js/'));
});

gulp.task('test-ui', function() {
    return gulp.src('./idontexist') // https://github.com/lazd/gulp-karma/issues/9 
        .pipe(karma({
            configFile: TEST_PATH + 'karma.conf.js'
        }))
        .on('error', handleError);
});
```

And a Karma configuration file, **karma.conf.js**:
```js
module.exports = function(config) {
  config.set({
    basePath : '.',
    frameworks: ['jasmine', 'commonjs'],
    files: [
        JS_PATH + 'libs/angular/angular.js',
        JS_PATH + 'libs/angular-mocks/angular-mocks.js',
        JS_PATH + 'services/*.js',
        JS_PATH + 'controllers/*.js',
        JS_PATH + 'directives/*.js',
        JS_PATH + 'app.js',
        TEST_PATH + '/**/*-spec.js'
    ],
    preprocessors: {
        JS_PATH + 'app.js': ['commonjs', 'coverage'],
        JS_PATH + 'libs/angular/angular.js': ['commonjs'],
        JS_PATH + 'libs/angular-mocks/angular-mocks.js': ['commonjs'],
        JS_PATH + 'services/*.js': ['commonjs', 'coverage'],
        JS_PATH + 'controllers/*.js': ['commonjs', 'coverage'],
        JS_PATH + 'directives/*.js': ['commonjs', 'coverage'],
        TEST_PATH + '/**/*-spec.js': ['commonjs']
    },
    reporters: ['coverage', 'progress'],
    coverageReporter: {
        type: 'lcov',
        dir: 'path/to/report',
        file: 'lcov.info'
    },
    port: 9876,
    runnerPort: 9100,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['PhantomJS'],
    captureTimeout: 60000,
    singleRun: true
  });
};
```

Only include source files that you want to generate coverage for in the `preprocessors` block. Do not include tests or libraries for coverage. Optionally, you can configure the reporter type that will be instrumented by [Istanbul](https://github.com/gotwarlost/istanbul) in the `coverageReporter` block.

### SonarQube
If the report type selected is LCOV, you can then import the resulting coverage reports into [SonarQube](http://www.sonarqube.org/) for *Continuous Inspection*, raising code quality visibility for all stakeholders and making it an integral part of the software development lifeycle.

Assuming that you have a SonarQube server [installed and running](http://www.sonarqube.org/screencasts2/installation-of-sonar/), the SonarQube [Javascript plugin](http://docs.sonarqube.org/display/SONAR/JavaScript+Plugin) can be used to:
-  run an analysis of your JavaScript project with the [SonarQube Runner](http://docs.sonarqube.org/display/SONAR/Analyzing+with+SonarQube+Runner)
-  display code coverage data by importing LCOV reports while running an analysis

##### Run analysis
Create a configuration file in the root directory of the project: **sonar-project.properties**
```properties
# First configure general information about the environment

# Project properties
sonar.projectKey=projectkey
sonar.projectName=Project Name
sonar.projectVersion=1.0

sonar.sources=path/to/src
sonar.language=js
sonar.sourceEncoding=UTF-8

# To import the LCOV report
sonar.javascript.lcov.reportPath=path/to/report/lcov.info
sonar.exclusions=file:path/to/libs/**/*.js
```

Then run the following command from the root directory in your Terminal or CI set-up:
```sh
$ sonar-runner
```

### Integration testing in Node.js

On the server, I use [Mocha](http://mochajs.org/) to test the **Node.js** RESTful API as it gives developers a lot of flexibility. On the other hand, Mocha may take more time to configure while you find the right choices for your application.

#### Mocha
- **Assertions**. Mocha allows any assertion library. In the example below I use [should.js](https://github.com/visionmedia/should.js) with its declarative, BDD style. [Chai](http://chaijs.com/) also enables the should style.
- **Asynchronous**. Easy to test asynchronous code. Simply invoke the `done()` callback or return a promise
- **Spies, stubs and mocks**. [Sinon.JS](http://sinonjs.org/) works with any testing framework
- **Code coverage**. Mocha can be set up along with **Istanbul** to generate coverage reports
- **[Supertest](https://github.com/visionmedia/supertest)** – Great library for testing Express-based RESTful endpoints


For instance, to test a Node.js RESTful API add the following in your **Gulpfile.js** after installing your dependencies:
```js
var gulp = require('gulp'),
    mocha = require('gulp-mocha'),
    istanbul = require('gulp-istanbul');

// Run Node.js tests and create LCOV reports with Istanbul
gulp.task('test-server', function () {
  return gulp.src([JS_PATH_SERVER + '**/*.js'])
      .pipe(istanbul()) // Node.js source coverage
      .on('end', function () {
          gulp.src(TEST_PATH_SERVER + '**/*.js')
              .pipe(mocha({
                  reporter: 'spec'
              }))
              .on('error', handleError)
              .pipe(istanbul.writeReports('path/to/report')); // Creating reports
      });
});
```

Then, your **Node.js** integration test files may look like the following:
```js
var should = require('should'),
    request = require('supertest');

it('should respond with a successful leaderboard array', function(done) {
    request(app)
        .get('/api/leaderboard')
        .set('Accept', 'application/json')
        .expect('Content-Type', 'application/json; charset=utf-8')
        .expect(200)
        .end(function(err, res) {
            expect(err).to.not.exist;
            res.should.have.status(200);
            res.body.should.be.array;
            done();
        });
});
```
Once this task has been run, just repeat the same process described in the [SonarQube](#sonarqube) section and **run a SonarQube analysis** for Node.js to complete the end-to-end **system testing strategy**.

### Protractor
If you want to go the extra mile, [Protractor](https://github.com/angular/protractor) enables end-to-end testing for AngularJS applications, running tests against your application in a real browser, interacting with it as a user would.

Setting up a testing strategy with Protractor goes beyond the scope of this article but in summary Protractor:
- uses **Jasmine** for its testing interface
- requires an instance of a **Selenium Server** running to control browsers
- needs two files to run, a **spec file** and a **configuration file**.

A spec file would look like the following:
```js
describe('Protractor demo', function() {
  it('should add one and two', function() {
    browser.get('http://juliemr.github.io/protractor-demo/');
    element(by.model('first')).sendKeys(1);
    element(by.model('second')).sendKeys(2);

    element(by.id('gobutton')).click();
    expect(element(by.binding('latest')).getText()).toEqual('3');
  });
});
```
Where the `describe` and `it` syntax is from the Jasmine framework. `browser` is a global created by Protractor, which is used for browser-level commands such as navigation.

With the combination of these techniques in conjunction with a CI set-up companion, you can automate a complete **system testing strategy** for your project, while increasing code quality visibility through a Continuous Inspection process.
