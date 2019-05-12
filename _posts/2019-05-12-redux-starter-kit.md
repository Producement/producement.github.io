---
layout: post
title: 'Reducing boilerplate with Redux Starter Kit'
author: maido
categories: [tech, react, redux]
image: /assets/images/reduxkit.png
featured: false
hidden: false
---

Every time we start a new project we do a retrospective and look back at our previous projects and try to evaluate which technical choices were good and where we might have gone wrong. Usually we are pretty happy with our [Kotlin](https://kotlinlang.org/) + [Spring Boot](https://spring.io/projects/spring-boot) + [Postgres](https://www.postgresql.org/) based backend, but at least I personally have constantly re-evaluated [React](https://reactjs.org/)/[Redux](https://redux.js.org/): is this the best tool we can use or is there something better?

Don't get me wrong, I like React, but my issue with it mostly boils down to the boilerplate you need to write to handle your React/Redux app state when asynchonously consuming your backend API: every simple call needs a state object, Action, Action Creator, Reducer. If you are using [TypeScript](https://www.typescriptlang.org) as we do it gets even harder because everything needs to be typed as well.

Luckily after some searching (and even creating a PoC using [Vue](https://vuejs.org/)+[Vuex](https://vuex.vuejs.org/) to evaluate alternatives) I stumbled upon [Redux Starter Kit](https://redux-starter-kit.js.org/), which promises to do to Redux what [CRA](https://facebook.github.io/create-react-app/) is doing to new React app creation: making it as simple as possible by being a bit opinionated. 
I'm not going to post and example without Redux Starter Kit, if you have used React/Redux you probably have a lot of them anyway, so I'm just going to show you how one slice of state can be set up using helpers provided by Redux Starter Kit.

Let's say we want to fetch a list of users from the backend and display them in a table or whatever.

First we set up our state.

```
import {User} from 'model';

export interface UsersState {
  users: User[];
  isLoading: boolean;
  error: string | null;
}

export const initialState: UsersState = {
  users: [],
  isLoading: false,
  error: null,
};
```

Then create a [slice](https://redux-starter-kit.js.org/api/createslice) (reducer with actions)

```
import {UserState, initialState} from 'state';
import {createSlice} from 'redux-starter-kit';

const usersSlice = createSlice<UsersState>({
  initialState,
  reducers: {
    fetchUsers: (state, action) => {
        state.isLoading = true;
    },
    fetchUsersSuccess: (state, action) => {
      state.users = action.payload.content;
      state.isLoading = false;
    },
    fetchUsersError: (state, action) => {
      state.error = action.payload;
      state.isLoading = false;
    },
  },
});

// Extract the action creators object and the reducer
const {actions, reducer} = usersSlice;
// Extract and export each action creator by name
export const {fetchUsers, fetchUsersSuccess, fetchUsersError} = actions;

export default reducer;
```

Combine reducers and create a store

```
import userReducer from 'reducer';
import { createStore, combineReducers } from 'redux';
const mainReducer = combineReducers({
  user: userReducer,
});

const store = createStore(mainReducer);
```

Use whichever [async middleware](https://redux.js.org/advanced/async-flow) you want to fetch the data. For example [redux-thunk](https://github.com/reduxjs/redux-thunk) or [redux-saga](https://redux-saga.js.org/). Using **redux-thunk** here.

```
import {userApi} from 'userApi';
import {PayloadAction, Action} from 'redux-starter-kit';

export const fetchUsers = () =>
  async (dispatch: Dispatch<PayloadAction>): Promise<Action> => {
    dispatch(fetchUsers());
    try {
      const users = await userApi.findAllUsers();
      return dispatch(fetchUsersSuccess(users));
    } catch (e) {
      return dispatch(fetchUsersError(e.message));
    }
  };
```

And now use it in your awesome functional connected React/Redux component (with [Hooks](https://reactjs.org/docs/hooks-intro.html) and TypeScript of course):

```
import React, {useEffect} from 'react';
import {connect, ResolveThunks} from 'react-redux';
import {fetchUsers as fetchUsersAction} from './userActions';
import {AppState} from '../store/store';

export const Users = ({users, fetchUsers}: StateProps & DispatchProps) => {

  useEffect(() => {
    fetchUsers();
  }, [fetchUsers]);

  return (
    <div className='container'>
      {users.length > 0 //Do something...}
    </div>
  );
};

const mapStateToProps = ({users}: AppState) => ({
  ...users,
});

const mapDispatchToProps = {fetchUsers: fetchUsersAction};

type StateProps = ReturnType<typeof mapStateToProps>;

type DispatchProps = ResolveThunks<typeof mapDispatchToProps>;

export default connect(mapStateToProps, mapDispatchToProps)(Users);
```

Done! 

YMMV but it definitely swayed us back to using React for our current and future projects! 
We will not stop evaluating other technologies whenever we start new projects, but for now we have more confidence in using React/Redux and delivering value instead of spending our time writing boilerplate code.