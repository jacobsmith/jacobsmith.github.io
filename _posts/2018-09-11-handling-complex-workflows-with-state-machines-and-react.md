---
layout: post
title: Handling complex workflows with State Machines and React
date: 2018-09-11 13:03 -0400
---

I recently worked on a site that had the requirement to import a CSV of records. A user could select what type of records it was, add a file, get a preview of what would be imported, then finally confirm the import. There were also some loading states in between as well.

Because there were so many different states that all needed to live (or at least be orchestrated) in one component, a state machine was a great way to keep everything on track and reduced the complexity of the code.

> A state machine is an entity that allows transitions between predefined states. An invalid transition between states will usually raise an error.
>
> 
>
> For example:
>
> Wake Up > Shower > Get Dressed is a proper transition
>
> Wake Up > Get Dressed > Shower would raise an error because the user is performing steps out of order.

A very basic example of a state machine is as follows:

```js
// state is the current state of the StateMachine
// states is all possible states of the state machine and defines the manner in which they can transition between states

class StateMachine {
    constructor(options) {
        this.state = options.initialState || options.states[0];
        this.states = options.states;
    }
    
    transitionTo(newState, metadata = null) {
      const currentState = this.state;
      const validNextStates = this.states.find(state => {
        return Object.keys(state)[0] === currentState;
      })[currentState].transitionsTo;

    if (validNextStates.indexOf(newState) > -1) {
      const stateMachine = new StateMachine({ states: this.states });
      stateMachine.state = newState;
      stateMachine.stateMetadata = metadata;

      return stateMachine;
    } else {
      throw new Error(`Invalid transition from ${currentState} to ${newState}`);
    }
  }
}
```



Let's deconstruct this quickly to see how it works:



To begin, we accept a set of states that define:

1. What the possible states are for this machine
2. What are valid transitions between those states

We then have a `transitionTo` method that accepts `newState` and `metadata` arguments. The first thing we do is get a handle to the current state and all valid transition states from our current state. We then are able to see if the desired `newState` is a valid transition from our current state in the machine. If it is not, we raise an error that can be caught or handled elsewhere. This truly should be an "exceptional" case as generally state machines have clearly defined (and tested) ways of transitioning, so we shouldn't get any surprises here.

If the transition is valid, we actually create a new instance of the `StateMachine` object, set its `states` collection to be accurate, and then update the current state. The primary reason for this is to ensure that our state machine integrates nicely with React's `setState` lifecycle - we don't want to directly mutate the `stateMachine` object on the component's `state` - instead, we can simply replace it with a new version that has the proper settings.

It may be possible to directly mutate the `stateMachine` and have React update properly - however, the cost of duplicating the `StateMachine` object pales in comparison to the cost of debugging component lifecycle errors caused by accidentally mutation state in a way that is not supported by React. To this end, I prefer to be a little more explicit, and write a little more code that is easier to understand and debug later, than try to eek out every ounce of performance. Your needs may vary, but for the vast majority of use cases, this is a good balance of speed and developer happiness.



The CSV states might defined as follows:

```js
const csvStates = {
  fileSelect: "fileSelect", // choose the file to upload
  uploading: "uploading", // upload the file
  showPreview: "showPreview", // show what will be imported
  importing: "importing", // actually import the records
  imported: "imported", // show user records were imported
  error: "error" // an error ocurred
};

const csvStateMachineOptions = {
  initialState: csvStates.fileSelect,
  states: {
    fileSelect: {
      transitionsTo: [csvStates.uploading, csvStates.error]
    },
    uploading: {
      transitionsTo: [csvStates.showPreview, csvStates.error]
    },
    showPreview: {
      transitionsTo: [csvStates.importing, csvStates.error]
    },
    importing: {
      transitionsTo: [csvStates.imported, csvStates.error]
    },
    imported: {
      transitionsTo: [csvStates.fileSelect, csvStates.error]
    },
    error: {
      transitionsTo: [csvStates.fileSelect]
    }
  }
};

export { csvStates, csvStateMachineOptions };

```

Now, for the actual component:

```js
import React, { Component } from "react";
import { csvStates, csvStateMachineOptions } from "./csvStateMachineOptions";
import StateMachine from "./stateMachine";
import {
  FileSelect,
  Uploading,
  ShowPreview,
  JobStatusPercent,
  ImportSuccessful,
  ErrorComponent
} from "./stages.js";

class ImportCSV extends Component {
  constructor() {
    super();
    this.state = {
      stateMachine: new StateMachine(csvStateMachineOptions)
    };
  }

  setStateMachine(newState, metadata) {
    this.setState({
      stateMachine: this.state.stateMachine.transitionTo(newState, metadata)
    });
  }

  render() {
    const currentState = this.state.stateMachine.state;
    let componentToRender = {
      props: {
        onError: err => {
          this.setStateMachine(csvStates.error, err);
        }
      }
    };

    switch (currentState) {
      case csvStates.fileSelect:
        componentToRender.component = FileSelect;
        componentToRender.props.onSelect = () => {
          this.setStateMachine(csvStates.uploading);
        };
        break;
      case csvStates.uploading:
        componentToRender.component = Uploading;
        componentToRender.props.onComplete = () => {
          this.setStateMachine(csvStates.showPreview);
        };
        break;
      case csvStates.showPreview:
        componentToRender.component = ShowPreview;
        componentToRender.props.importCsv = () => {
          this.setStateMachine(csvStates.importing);
        };
        break;
      case csvStates.importing:
        componentToRender.component = JobStatusPercent;
        componentToRender.props.onComplete = jobData => {
          this.setStateMachine(csvStates.imported, { data: jobData });
        };
        break;
      case csvStates.imported:
        componentToRender.component = ImportSuccessful;
        componentToRender.props.onConfirm = () => {
          this.setStateMachine(csvStates.fileSelect);
        };
        break;
      case csvStates.error:
        componentToRender.component = ErrorComponent;
        componentToRender.props.error = this.state.stateMachine.stateMetadata;
        componentToRender.props.startOver = () => {
          this.setStateMachine(csvStates.fileSelect);
        };
        break;
      default:
        console.log("Unhandled state: ", currentState);
        return null;
    }

    return React.createElement(
      componentToRender.component,
      componentToRender.props
    );
  }
}

export default ImportCSV;

```

As you can see, the component itself is relatively small and compact - the largest amount of code is simply delegating to other, more specialized components to render each step in the process.

One thing to note is that we are assuming every component takes an `onError` function to report when something went wrong with its step of the process. Rather than duplicate that `onError={ (err) => this.transitionTo(csvStates.error, err) }` on every component, we instead make use of `React.createElement` to create each element based on a config object we create in the render section. This reduces code duplication and also allows us to change the API for the `onError ` function in one place in the future if we want (change from a string to an object, etc.).

Overall, utilizing a state machine can reduce the complexity of your code and provide an easy-to-follow delineation between discrete steps in your application.

Each of the "step" components (`FileSelect`, `Uploading`, etc.) are stubbed out as simple views with a button progressing to the next step in the state machine. You can play with a fully working example on CodeSandbox below.

[![Edit jp2vx3lk23](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/jp2vx3lk23)


