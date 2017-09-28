At my employer, my team recently made a decision to integrate VueJS in to a Rails application (thankfully, we had kept it upgraded and were on Rails 5.1) that we're developing.

We landed on this decision because we discovered that we had some fairly complex/complicated UI/UX requirements for a few section of the application and wanted to avoid hand-writing lots of jQuery and DOM manipulation code to facilitate those UI pieces.

So far, it's been a great success. There was one thing that I wanted to document, though, that I spent quite a bit of time working on: integrating VueJS/JavaScript specs in to the application.

# Set up

First things first, I chose [mocha](https://mochajs.org/) and [chai](http://chaijs.com/) as our JavaScript testing framework. I wanted something that would be familiar to the other developers on my team, who have years of experience writing RSpec tests, so that they could jump right in and start being immediately productive.

With that out of the way, I set out to determine how to actually test a VueJS app. I quickly came across an excellent blog post ([unit testing VueJS components for beginners](https://eddyerburgh.me/unit-test-vue-components-beginners)) by [eddyerburgh](https://github.com/eddyerburgh). From here, I was able to scoop up [avoriaz](), which is eddyerburgh's VueJS unit testing utility library, and  [jsdom](), which gives us a minimal browser environment (sort of, you can read more about it on the GitHub page) to load the VueJS app in to.

## Dependencies

```bash
$ yarn add mocha chai mocha-webpack avoriaz jsdom jsdom-global --save-dev
```

# Specs

## Runner

One of the first things you'll run up against in this scenario is integrating the testing framework in to the applications ecosystem. In our case, we're using `yarn` to manage our JavaScript dependencies and `webpacker` to manage our JavaScript assets. We still use `sprockets`, too, for some JavaScript in the "older" style. But, that's besides the point. In order to be able to execute the test suite, we need to set a test runner up. This sets the environment (very important -- it took me a bit of time and googling to sort this piece out) and gets all of the required bits passed in to `mocha-webpack`.

```json
// add the following to your package.json

"scripts": {
    "test": "NODE_ENV=test mocha-webpack --webpack-config config/webpack/test.js spec/javascript/**/*.spec.js --recursive --require spec/javascript/setup.js"
}
```

This makes a couple of assumptions:

* You used the Rails webpacker generators to install VueJS (which gives you the configuration chain in `config/webpacker.yml` and `config/webpack/**/*`
* Your mocha/chai specs live in `spec/javascript`

You'll also need to deal with initializing `jsdom`, add the following to a file in `spec/javascript/setup.js`:

```javascript
require('jsdom-global')()
```

With this configured, you can execute the test suite with the command: `$ yarn test`.

# avoriaz

The last thing I want to mention is how amazing avoriaz is for helping with VueJS tests. First off, the [documentation](https://eddyerburgh.gitbooks.io/avoriaz/content/) is high quality. Secondly, it let's you very easily and quickly access a VueJS and it's components in a clear way with it's `mount()` method:

```javascript
import { mount } from 'avoriaz';
import Foo from './Foo.vue';

const wrapper = mount(Foo);

expect(wrapper.text()).to.equal('some text');
```

We are using object literal stores in our VueJS applications at the moment (we haven't yet needed to reach for [VueX](https://vuex.vuejs.org/en/)) and avoriaz makes accessing it as easy as:

```javascript
import { expect } from 'chai'
import { mount } from 'avoriaz'

import SomeVueJsApp from '../../app/javascript/some_vue_js/app.vue'

describe('SomeVueJsApp', () => {
    const wrapper = mount(SomeVueJsApp)

    it('exposes the applictions store', () => {
        expect(wrapper.data().store.state.users.length).to.equal(2)
    })
})
```

It's also easy to pass props in to the mount() method like:

```javascript
const characters = ['a', 'b', 'c']
const numbers = [1, 2, 3, 4]

const wrapper = mount(SomeVueJsApp, { propsData: { characters, numbers }})
```

These are obviously convoluted examples but the point remains, this is actually very easy to use and quite powerful. I think that's all that I had to say about this for now. I mostly wanted to document it for my own posterity, since it took me an hour or so to piece these things together. It might save me time again in the future or someone else.
