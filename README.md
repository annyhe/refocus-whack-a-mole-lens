# Build a whack-a-mole game with Refocus!
Let's have some fun!
What we'll cover in this tutorial:
  - LDK Setup
  - stream mock data in LDK
  - DOM manipulations on realtime data change
  - implement a game with refocus!

`Insert a screenshot of the whack a mole game`.

### Setup

The refocus ldk requires [Node.js](https://nodejs.org/) v4+ to run.

Clone the LDK, install the dependencies, then create a directory for the new lens.
```sh
$ git clone https://github.com/Salesforce/refocus-ldk
$ cd refocus-ldk
$ npm install
$ mkdir Lenses/SimpleLens/src
```
Add the default lens template.
```sh
$ cp main.template Lenses/SimpleLens/src/main.js
```
Set the LDK to use the new lens:
```sh
npm config set refocus-ldk:lens SimpleLens
```
Open two terminal tabs: run
```sh
npm run prototype
```
to start up server. Now you're ready to see the events stream in!
Go to http://localhost:3000/configForm.html to set how often to send in the realtime events.
`insert screenshot of the config events page`.

On clicking the Save button, you will be redirected to the lens page on http://localhost:3000.
Open the browser console to see the streaming events being logged:  `insert screenshot of the events being streamed in`.

##### Add a dataset
For this game we need a 6 by 6 dataset. We achieve this with 6 subjects, and 6 aspects.
Add the subjects dataset to `public/dataset/one-level.json`
```json
{
  "absolutePath": "Fellowship",
  "name": "Fellowship",
  "samples": [],
  "children": [
    {
      "absolutePath": "Fellowship.Gandalf",
      "name": "Gandalf",
      "samples": [],
    },
    {
      "absolutePath": "Fellowship.Boromir",
      "name": "Boromir",
      "samples": [],
    },
    {
      "absolutePath": "Fellowship.Aragorn",
      "name": "Aragorn",
      "samples": [],
    },
    {
      "absolutePath": "Fellowship.Peregrin",
      "name": "Peregrin",
      "samples": [],
    },
    {
      "absolutePath": "Fellowship.Meriadoc",
      "name": "Meriadoc",
      "samples": [],
    },
    {
      "absolutePath": "Fellowship.CHX",
      "name": "CHX",
      "samples": [],
    }
  ]
}
```

Add the aspects dataset to `public/dataset/one-level-aspects.json`
```json
["and", "she", "said", "winter", "is", "coming"]
```


Add the path to the dataset as another `<option>` element in `public/configForm.html`
Go to http://localhost:3000/configForm.html to change to use the newly added dataset.



### Set up mock data
We still need to provision the data. You can choose which dataset to use from the dropdown on http://localhost:3000/configForm.html
`insert screenshot of the config dropdown`.
For this tutorial we choose the `Flat hierarchy Simulation` dataset from the dropdown

### Change data stream
To change the format of the data being streamed in, go to `src/Realtime.js`.
We need to change the random data generation to return only subjects that is one level deep, since subjects make up the columns of the whack a mole game. We also need to limit the aspects to only those defined in the dataset, which correspond to the rows in our game.

The default `prepareSubjectAdds` generate subjects that are multiple levels deep. We need to change it to return only subjects defined in `./
public/datasets/one-level.json`

Replace the `prepareSubjectAdds` with
```js
prepareSubjectAdds() {
  const retval = [];
  const x = Random.intBetween(0, this.conf.subjects.maxAdd);
  console.log('ldk.Realtime.prepareSubjectAdds', new Date(),
    `Preparing a batch of ${x} subject adds`);
  // const subjectValues = d3c.values(this.inventory.subjects);
  if (x) {
    [...Array(x).keys()].forEach(() => {
      const idx = Math.floor(Math.random() * dataset.children.length);// between 0 and 10
      const subject = dataset.children[idx];
      retval.push({ 'subject.add': subject }); // a subject from the dataset
      this.addToInventory(subject);
    });
  }
  return retval;
} // prepareSubjectAdds
```

The default `prepareSampleAdds` generate samples with random aspects. We need to change it to return only aspects defined in `./public/datasets/one-level-aspects.json`

Replace the `prepareSampleAdds` with
```js
prepareSampleAdds() {
  const retval = [];
  const x = Random.intBetween(0, this.conf.samples.maxAdd);
  console.log('ldk.Realtime.prepareSampleAdds', new Date(),
    `Preparing a batch of ${x} sample adds`);
  const subjectValues = d3c.values(this.inventory.subjects);
  if (x && subjectValues.length) {
    [...Array(x).keys()].forEach(() => {
      const idx = Random.intBetween(0, subjectValues.length - 1);
      const subject = subjectValues[idx];
      const s = Random.addSample(subject);
      if (!this.inventory.samples[s.name]) {
        this.inventory.samples[s.name] = s;
        retval.push({ 'sample.add': s });
      }
    });
  }
  return retval;
} // prepareSampleAdds
```

Run this command to build and compile the ldk
```sh
npm run ldk-compile
```
Go to http://localhost:3000/configForm.html to set the config to stream in subject and sample add. Hit `Save`, open the console.log, and should see the subjects being sent in is only a level deep, as in the absolutePath only contains one `.` separator.

### Implement the realtime change handler
For our game, the data structure to store subjects and aspects is
```js
[   {name: 'subjectName',
    absolutePath: 'subjectAbsolutePath1',
    ...,
    samples: [{
        name: 'subjectAbsolutePath1|aspectName1',
        ...
        }, {
        name: 'subjectAbsolutePath1|aspectName2',
        ...
        }]
    }, {name: 'subjectName',
    absolutePath: 'subjectAbsolutePath2',
    ...,
    samples: [{
        name: 'subjectAbsolutePath2|aspectName3',
        ...
        }, {
        name: 'subjectAbsolutePath2|aspectName4',
        ...
        }]
    }, ...
]
```

When the page receives streamed in events, we need to update this data handler. We also include another utility function that illustrates how the data is stored.
Add the following file:
```js
/**
 * Lenses/SimpleLens/src/utils.js
 *
 * Utility functions to kep the client-side subject and sample data
 * updated with streamed-in data
 */

/**
 * Return the index of the subject in the subjectsData, and
 * the indexes of the subject's samples from aspectsData
 * @param {Object} subject with samples array
 * @param {Array} subjectsData All allowable subjects
 * @param {Array} aspectsData All allowable aspects
 * @returns {Object} { Integer_index, [ int_index1, int_index2, ... ]}
*/
function getIndexOfSubjectAndSample(subject, subjectsData, aspectsData) {
  let subjectIndex = -1;
  for (let i = subjectsData.children.length - 1; i >= 0; i--) {
    if (subjectsData.children[i].absolutePath === subject.absolutePath) {
      subjectIndex = i;
      break;
    }
  }

  const sampleIndexes = [];
  for (let i = subject.samples.length - 1; i >= 0; i--) {
    const aspectName = subject.samples[i].name.split('|')[1];
    sampleIndexes.push(aspectsData.indexOf(aspectName));
  }

  return { subjectIndex, sampleIndexes };
}

const realtimeChangeHandler = {
  onSampleAdd(sample, subjects) {
    const subjectPath = sample.name.split('|')[0];
    for (let subject of subjects) {
      subject.samples = subject.samples ? subject.samples : [];
      if (subject.absolutePath === subjectPath) {
        subject.samples.push(sample);
        break;
      }
    }
  },
  onSampleRemove(sample, subjects) {
    const subjectPath = sample.name.split('|')[0];
    for (let subject of subjects) {
      if (subject.absolutePath === subjectPath) {
        const index = subject.samples.indexOf(sample);
        subject.samples = subject.samples.filter((samp) => {
          return samp.name !== sample.name;
        })
        break;
      }
    }
  },
  onSampleUpdate(change, subjects) {
    let sample = change.new;
    const subjectPath = sample.name.split('|')[0];
    for (let subject of subjects) {
      if (subject.absolutePath === subjectPath) {
        subject.samples = subject.samples ? subject.samples : [];
        const sampleNames = subject.samples.map((samp) => {
          return samp.name;
        });
        // get sample by name
        const index = sampleNames.indexOf(sample.name);
        subject.samples[index] = sample;
        break;
      }
    }
  },
  onSubjectAdd(subject, subjects) {
    subjects.add(subject);
  },
  onSubjectRemove(subject, subjects) {
    subjects.delete(subject);
  },
  onSubjectUpdate(change, subjects) {
    const updatedSubject = change.new;
    // update the subject. keep the samples of the old
    let oldSubject = {};
    for (let subject of subjects) {
      if (subject.id === updatedSubject.id) {
        oldSubject = subject;
        break;
      }
    }

    const samples = oldSubject.samples;
    subjects.delete(oldSubject);
    updatedSubject.samples = samples;
    subjects.add(updatedSubject);
  }
}; // realtimeChangeHandler

module.exports = {
  realtimeChangeHandler,
  getIndexOfSubjectAndSample,
};

```
We also need HTML to display the lens and the score. For our simple lens we will use handlebars.
Make a file in `Lenses/SimpleLens/src/templates` called `gameSpace.handlebars` with the following
```html
<section>
    <p id='subjectsArr'></p>
    <div id='info'></div>
    <section id='stage'></section>
    <p>
        Score: <span id="score">0</span> points!
    </p>
</section>
```
We also need to render the game stage and update the scores. Make a file with the following:
```js
/**
 * Lenses/SimpleLens/src/game.js
 *
 * Specifies game-centric DOM manipulations
 */
function updateScore() {
  let score = parseInt(document.getElementById('score').innerHTML);
  document.getElementById('score').innerHTML = score+=1;
}

function renderStage(height, width) {
  for (let i = height; i > 0; i--) {
      const div = document.createElement('div');
      for (let j = width; j > 0; j--) {
          const p = document.createElement('p');
          p.onclick = function(event) {
            if (event.target.className ==='highlight') {
              updateScore();
              event.target.className = '';
            }
          };
          div.appendChild(p);
      }
      document.getElementById('stage').appendChild(div);
  }
}

module.exports = {
  updateScore,
  renderStage,
};
```

To give our game a spiffy look, make a css file with path `Lenses/SimpleLens/src/lens.css` with the following
```css
body {
  margin: 0;
    background-color: #ccc;
}

#stage div p.highlight {
    background: yellow;
}

#lens {
    margin: 50px;
}
#stage div {
  display: inline-block;
}

#stage p  {
  color: white;
    background-color: #ccc;
    display: block;

    height: 50px;
    margin: 0 0 5px 5px;
    text-decoration: none;
    width: 50px;

    background-color: #4f4f4f;
    width: 40px;
    height: 40px;
    margin: 5px;
    border-radius: 20px;
    box-shadow: inset 0 3px 4px rgba(0,0,0,.5),
    inset 0 0 10px rgba(0,0,0,.5),
    0 0 2px 2px rgba(0,0,0,.15),
    0 -1px 2px 1px rgba(0,0,0,.25),
    0 2px 3px 2px rgba(255,255,255,.5),
    0 -2px 10px 1px rgba(255,255,255,.75),
    0 2px 10px 3px rgba(0,0,0,.25);
}
```
Lastly, we combine all our hard work into a single file. This file sets up the realtime event handlers and calls other files to set up the DOM.
Add the `Lenses/SimpleLens/src/main.js`
```js
/**
 * Lenses/SimpleLens/src/main.js
 *
 * Sets up event handlers and DOM manipulations in response to streamd-in data.
 */
'use strict';
require('./lens.css');
const subjectsData = require('../../../public/datasets/one-level.json');
const aspectsData = require('../../../public/datasets/one-level-aspects.json');
const loadingTemplate = require('./template/gameSpace.handlebars');
const renderStage = require('./game').renderStage;
const realtimeChangeHandler = require('./utils').realtimeChangeHandler;
const getIndexOfSubjectAndSample = require('./utils').getIndexOfSubjectAndSample;
const LENS_ELEMENT = document.getElementById('lens');
const subjects = new Set(); // set of objects. each subject contains an array of samples

/**
 * Perform your DOM manipulation here.
 */
function draw() {
  console.log(new Date(), 'drawing');
  const STAGE = document.getElementById('stage');
  const infoHolder = document.getElementById('info');
  for (let subj of subjects) {
    const { subjectIndex, sampleIndexes } = getIndexOfSubjectAndSample(subj, subjectsData, aspectsData);
    // info.innerHTML = 'subject ' + subj.absolutePath + ', is at index ' +
    //   subjectIndex + ' has ' + subj.samples.length + ' samples' + ' and they are at ' + sampleIndexes;
    for (let i = sampleIndexes.length - 1; i >= 0; i--) {
      // highlight samples
      const sampleNode = STAGE.childNodes[subjectIndex].childNodes[sampleIndexes[i]];
      sampleNode.className = 'highlight';
      window.setTimeout(function(){
        // remove class after a time period
        sampleNode.className = '';
      }, 1000); //<-- Delay in milliseconds
    }
  }

  // document.getElementById('subjectsArr').innerHTML = '# of Columns:' + subjects.size + ', ' + [...subjects].map((subj) => subj.name);
} // draw

const eventTarget = {
  document: document,
  lens: LENS_ELEMENT,
  window: window,
};

/**
 * All the event handlers configured here will be registered on
 * refocus.lens.load.
 */
const eventHandler = {
  lens: {

    /**
     * Handle the refocus.lens.realtime.change event. The array of changes is
     * stored in evt.detail. Iterate over the array to perform any preprocessing
     * if needed, then call draw only once after all the data manipulation is
     * done.
     */
    'refocus.lens.realtime.change': (evt) => {
      console.log(new Date(), 'refocus.lens.realtime.change',
        'contains ' + evt.detail.length + ' changes');
      if (!Array.isArray(evt.detail) || evt.detail.length == 0) {
        return;
      }

      evt.detail.forEach((chg) => {
        if (chg['sample.add']) {
          realtimeChangeHandler.onSampleAdd(chg['sample.add'], subjects)
        } else if (chg['sample.remove']) {
          realtimeChangeHandler.onSampleRemove(chg['sample.remove'], subjects)
        } else if (chg['sample.update']) {
          realtimeChangeHandler.onSampleUpdate(chg['sample.update'], subjects);
        } else if (chg['subject.add']) {
          realtimeChangeHandler.onSubjectAdd(chg['subject.add'], subjects)
        } else if (chg['subject.remove']) {
          realtimeChangeHandler.onSubjectRemove(chg['subject.remove'], subjects)
        } else if (chg['subject.update']) {
          realtimeChangeHandler.onSubjectUpdate(chg['subject.update'], subjects)
        }
      }); // evt.detail.forEach

      // Now that we've processed all these changes, draw!
      draw();
    }, // refocus.lens.realtime.change
  },
}; // eventHandler

/*
 * Handle the load event. Register listeners for interesting window and lens
 * events here. This would also be a good place to render any DOM elements
 * which are not dependent on the hierarchy data (e.g. page header, page
 * footer, legend, etc.).
 */
LENS_ELEMENT.addEventListener('refocus.lens.load', () => {
  console.log(new Date(), '#lens => refocus.lens.load');
  LENS_ELEMENT.innerHTML = loadingTemplate();
  renderStage(subjectsData.children.length, aspectsData.length);

  // Register all the event listeners configured in eventHandler.
  Object.keys(eventHandler).forEach((target) => {
    Object.keys(eventHandler[target]).forEach((eventType) => {
      eventTarget[target].addEventListener(eventType, eventHandler[target][eventType]);
    });
  });
}); // "load" event listener
```
That's it for the files we need! Compile the lens by running these on separate tabs
```sh
$ npm run compile-watch
```

And make sure the server is running
```sh
$ npm run prototype
```
Go to http://localhost:3000/configForm.html to configure subjects and samples to streame in.
Hit `Save` and enjoy your whack a mole game at http://localhost:3000!


### Buid and deploy
Add a lens description json file
```sh
$ touch Lenses/SimpleLens/src/lens.json
```
Paste the following:
```json
{
  "name": "SimpleLens",
  "description": "A basic lens with realtime listeners",
  "helpEmail": "YOUR_EMAIL_HERE",
  "helpUrl": "YOUR_URL_HERE",
  "license": "YOUR_LICENSE",
  "version": "0.0.1"
}
```
Package up all your lens resources into `./dist/SimpleLens.zip` for deployment using the build script:
```sh
npm run build
```
For deployment, when you are ready to install your new lens into Refocus, upload `./dist/SimpleLens.zip` as your lens library file use the Refocus API `/v1/lenses`.

### Development
We welcome contributions in the form of new lenses, LDK improvements, and whatever else you can think of!
The code for this is up at


##### Add lens tests
Add tests in the test folder. Then run
```sh
$ npm run test
```


Next Steps
===
- implement different scores for different realtime events
- build a lens using multi-level data
