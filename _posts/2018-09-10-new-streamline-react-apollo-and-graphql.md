---
layout: post
title: Streamline React Apollo and GraphQL
date: 2018-09-10 17:28 -0400
---
Streamline React Apollo and GraphQL

React Apollo seems to be the de facto method of querying GraphQL endpoints from within the React ecosystem. Their Query and Mutation components integrate easily in to a developer's mental model of how React works. However, there are a few ways that we can simplify these components to handle 80% of our use cases with less code.

Please note that this does not intend to solve every problem you may encounter with Apollo - rather, it reduces some of the options available to you to get you back to code as soon as possible. There will still be times when you need the full API - these tricks are just for reducing boilerplate when that level of customization is not needed.

### TL;DR

React components that abstract away error handling and loading so you can focus
on the parts that are unique to your component.

Example:

```js
import React from 'react';
import Query from 'src/components/query';
import MovieListEntry from './movieListEntry';

const MoviesList = () => (
  <Query query={ movies }>
    {
      ({ movies }) => movies.map((movie) => {
        return <MovieListEntry movie={ movie } />;
      })
    }
  </Query>
);

export default MoviesList;
```

We can similarly do the same with `Mutation`:

```js
import React from 'react';
import Mutation from 'src/components/mutation';
import AfterMutation from 'src/components/afterMutation';

const LikeMovie = ({ movieId }) => (
  <Mutation mutation={ likeMovieMutation }>
    {
        (likeMovie, data) => {
            return (
              <button onClick={ likeMovie({ id: movieId }) }>Like Movie!</button>
            );
        }
    }

    <AfterMutation>
      Yay! You already liked this movie!    
    </AfterMutation>
  </Mutation>
);

export default LikeMovie;
```

Interested in what it takes to make these things possible? Keep reading!

## Building these components

To begin, we will create our own Query component:

```js
// src/components/query.js

import React from 'react';
import { Query as ApolloQuery } from 'react-apollo';

const Query = ({ query }) => (
  <ApolloQuery query={ query }>
    {
      ({ data, error, loading }) => {
        return children(data);        
      }
    }  
  </ApolloQuery>
);

export default Query;
```

Now, as you may very astutely point out, we're not actually handing loading or errors at all (and this will probably raise an error unless our children function handles undefined for its argument). To fix that, we want to handle all error and loading states in the same way. These might be a spinner icon for Loading and a custom alert modal for errors, inline errors on a form, etc. Whatever you want it to be, you can customize it easily.

```js
const LoadingAndErrorIndicator = ({ loading, error, children }) => {
    if (loading) {
        return <div>Loading...</div>;
    }
    
    if (error) {
        return <div className='error'>{ error }</div>
    }
    
    return children;
}
```

And now we can incorporate it like so:

```js
// src/components/query.js

import React from 'react';
import { Query as ApolloQuery } from 'react-apollo';
import LoadingAndErrorIndicator from './loadingAndErrorIndicator.js';

const Query = ({ query, ...rest }) => (
  <ApolloQuery query={ query } ...rest >
    {
      ({ data, error, loading, refetch }) => (
        <LoadingAndErrorIndicator error={ error } loading={ loading }>
          { children(data, ...data, { refetch }) }
        </LoadingAndErrorIndicator>
      )
    }  
  </ApolloQuery>
);

export default Query;
```

You'll also note that we added the refetch function as another argument to the children function. It's not necessary, but occasionally comes in useful if you need to explicitly rerun a query.

At this point, we can write a component to do the following:

```js
import React from 'react';
import Query from 'src/components/query';
import MovieListEntry from './movieListEntry';

const MoviesList = () => (
  <Query query={ movies }>
    {
      ({ movies }) => movies.map((movie) => {
        return <MovieListEntry movie={ movie } />;
      })
    }
  </Query>
);

export default MoviesList;
```

## Mutation

We can also do a similar thing with the Mutation component:

```js
import React from 'react';
import { Mutation as ApolloMutation } from 'react-apollo';

const Mutation = ({ mutation, children, ...rest }) => (
  <ApolloMutation mutation={ mutation } ...rest>
    {
        (mutationFn, { loading, error, data }) => (
          <LoadingAndErrorIndicator loading={ loading } error={ error } />
            { children(mutationFn, data) }
          </LoadingAndErrorIndicator>
        )
    }  
  </ApolloMutation>
);

export default Mutation;
```

This can be used like so:

```js
import React from 'react';
import Mutation from 'src/components/mutation';
import likeMovieMutation from 'src/mutations/likeMovie';

const LikeMovie = ({ movieId }) => (
  <Mutation mutation={ likeMovieMutation }>
    {
        (likeMovie, data) => {
            return (
              <button onClick={ likeMovie({ variables: { id: movieId }})}>Like Movie!</button>
            );
        }
    }
  </Mutation>
);

export default LikeMovie;
```

Now, this will execute the `likeMovie` function every time the button is pressed. But what do we do if we only want the user to click it once and then show a "You liked the movie!" message instead of the button?

To accomplish this, we will utilize an empty component - it will render only its children and have no markup of its own. So what does that get us, you might ask. It gives us a way for us to differentiate different nodes within the Mutation component - now we can replace one node with another after the mutation has taken place.

```js
import React from 'react';

const AfterMutation = ({ children }) => (
  children
);

export default AfterMutation;
```

And our Mutation file will be updated to be as follows:

```js
import React from 'react';
import { Mutation as ApolloMutation } from 'react-apollo';
import AfterMutation from './afterMutation';

// If you are using ReactHotReloader, you will need to compare against the following constant instead of AfterMutation.
// RHL wraps components to accomplish its goal, so normal type comparisons will not work
const AfterMutationType = <AfterMutation />.type;

const renderChildren = (children, mutationFn, data) {
    let afterMutation;
    
    React.Children.forEach(children, (child) => {
      if (child.type === AfterMutation) {
        afterMutation = child;
      }
    });
    
    if (afterMutation && data) {
        return afterMutation;
    }

    return children(mutationFn, data);
}

const Mutation = ({ mutation, children, ...rest }) => (
  <ApolloMutation mutation={ mutation } ...rest>
    {
        (mutationFn, { loading, error, data }) => (
          <LoadingAndErrorIndicator loading={ loading } error={ error } />
            { renderChildren(children, mutationFn, data) }
          </LoadingAndErrorIndicator>
        )
    }  
  </ApolloMutation>
);

export default Mutation;
```

So now, we can write something like this:

```js
import React from 'react';
import Mutation from 'src/components/mutation';
import AfterMutation from 'src/components/afterMutation';

const LikeMovie = ({ movieId }) => (
  <Mutation mutation={ likeMovieMutation }>
    {
        (likeMovie, data) => {
            return (
              <button onClick={ likeMovie({ variables: { id: movieId }})}>Like Movie!</button>
            );
        }
    }
    
    <AfterMutation>
      Yay! You already liked this movie!    
    </AfterMutation>
  </Mutation>
);

export default LikeMovie;
```

The last thing that I personally got tired of was writing out `{ variables: ... }` in the mutation functions. Is it petty and small? Probably. But it did feel much better to write `likeMovie({ id: movieId })` or, in another place, `likeMovie(...this.state)` (assuming we had a one-to-one mapping of state and the graphql variables).

To accomplish that change, we can do:

```js
const renderChildren = (children, mutationFn, data) {
    let afterMutation;
    
    React.Children.forEach(children, (child) => {
      if (child.type === AfterMutation) {
        afterMutation = child;
      }
    });
    
    if (afterMutation && data) {
        return afterMutation;
    }

    const newMutationFn = (variables) => {
        return mutationFn({ variables });
    }
    return children(newMutationFn, data);
}
```

That's all that's needed to stop having to write out `{ variables: ... }` in every mutation function. Obviously, this may not follow the element of least surprise, so it's one to think about before implementing. As a one-man team, I have a little more lattitude to do things like this and implement them. (I wouldn't throw this on a team without their approval before hand.)

Overall, the loading and error handling consistency alone has been well worth updating my existing apps.

I think GraphQL has some real benefits for teams moving forward. That said, it does bring its own pain points. Coming from a Rails background, I'm a big believer in streamlining the primary code paths while still offering the full API to handle things in a custom way if and when they need it. One of those areas for me was error handling and loading indicators. And while we were at it, being able to show a "post-mutation" message without doing custom comparisons in each component was a big win for me as well!
