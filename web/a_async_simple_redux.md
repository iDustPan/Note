
## Redux async

```
import { createStore, applyMiddleware } from 'redux';
import { createLogger } from 'redux-logger';
import axios from 'axios';
import thunk from 'redux-thunk';
import promise from 'redux-promise-middleware';

const initialState = {
    isFetching: false,
    fetched: false,
    comments: {},
    error: null
}

const rootReducer = (state = initialState, action) => {
    switch (action.type) {
        case "FETCH_COMMENTS_PENDING":
            state = {
                ...state,
                isFetching: true,
                fetched: false
            }
            break;
        case "FETCH_COMMENTS_FULFILLED":
            state = {
                ...state,
                isFetching: false,
                fetched: true,
                comments: action.payload.data
            }
            break;
        case "FETCH_COMMENTS_REJECTED":
            state = {
                ...state,
                isFetching: false,
                fetched: true,
                error: action.payload.response.data
            }
            break;
    }
    return state;
}

const store = createStore(rootReducer, applyMiddleware(promise(), thunk, createLogger()));

store.subscribe(() => {
    const state = store.getState()
    if (state.isFetching) {
        document.querySelector('.container').innerHTML = "loading...";
    }else if (state.fetched && !state.error && state.comments) {
        document.querySelector('.container').innerHTML = state.comments.total ;
    }else{
        document.querySelector('.container').innerHTML = state.error.messages[0];
    }
});
```

### use with `redux-promise-middleware`

```
store.dispatch({
    type: "FETCH_COMMENTS",
     payload: axios.get("https://baleen-dev.bybieyang.com/articles/article_cover_test/comments?f=0&t=20")
 });
```

 ### use with `redux-thunk`
 
```
// store.dispatch((dispatch) => {
//     dispatch({type: "FETCH_COMMENTS_PENDING"});
//     axios.get("https://baleen-dev.bybieyang.com/articles/article_cover/comments?f=0&t=20").then((response) => {
//         dispatch({type: "FETCH_COMMENTS_FULFILLED", payload: response.data});
//     }).catch((error)=>{
//         dispatch({type: "FETCH_COMMENTS_REJECTED", payload: error.response.data});
//     });
// })
```
