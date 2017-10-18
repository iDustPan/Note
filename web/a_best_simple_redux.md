```
import { createStore, applyMiddleware } from 'redux';
import { createLogger } from 'redux-logger'

const mathReducer = (state = 3, action) => {
    switch (action.type) {
        case "+":
            return state + action.payload;
        case "-":
            return state - action.payload;
    }
    return state;
}

const store = createStore(mathReducer, 3, applyMiddleware(createLogger()));

store.subscribe(() => {
    document.getElementById('container').innerHTML = store.getState();
});

store.dispatch({
    type: "+",
    payload: num
});

const subAction = (num) => {
    return {
        type: "-",
        payload: num
    };
}

store.dispatch(subAction(5));
```
