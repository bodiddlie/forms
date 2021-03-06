# forms

## Install

```sh
npm install --save [tbd]
```

## Basic Usage

Let's start by creating a basic login form. The `Form` component is your main entry-point to the API. It will create an HTML form and in this case, two input fields for email and password:

```jsx
import React from 'react'
import { Form, Field } from [tbd]

const LoginForm = props => (
  <Form>
    <Field name="email" component="input" />
    <Field name="password" component="input" type="password" />
  </Form>
)
```

The `Field` component is used to bootstrap state to the `Form` component. It always requires a `name` prop which is unique for the form. To designate what type of field will be created, pass a `component` prop with a string value for `input`, `textarea`, or `select`. If `input` is passed, you can also pass a `type` to get the desired HTML input type.

Since this API is mostly about providing transient state to you, it tries to have no opinions on design, validation, or submission. The above usage of `Form` and `Field`s will simply create this:

```html
<form>
  <input type="text" name="email" />
  <input type="password" name="password" />
</form>
```

At this point, the form will behave as expected for an HTML form regarding submission and validation (no validation at this time).


## Field Wraps

Generally speaking, you probably have some sort of consistent wrapping DOM around all your input fields. The specifics of yours may differ, but the concept is probably similar to this:

```html
<div class="field-wrap">
  <label>{label}</label>
  <div class="input">
    <input type="text">
  </div>
</div>
```

To achieve this without repeating the same HTML by hard-coding this around `Field`, create a custom version of `Field` like this:

```jsx
const FieldWrap = props => {
  const { label, type, ...rest } = props

  return (
    <Field {...rest}>
      {(inputProps, fieldState, formState) => (
        <div className="field-wrap">
          <label>{label}</label>
          <div className="input">
            <input {...inputProps} type={type} />
          </div>
        </div>
      )}
    </Field>
  )
}
```

In this case, our `FieldWrap` is returning `Field` and we're passing a function into `Field` to define the contents of our field-wrap logic. The name `FieldWrap` is not important as you may call yours whatever you want, or have many variations of it. This `FieldWrap` component can be used as follows:

```jsx
const LoginForm = props => (
  <Form>
    <FieldWrap label="Email" name="email" />
    <FieldWrap label="Password" name="password" type="password" />
  </Form>
)
```

Notice that by passing a function to `Field`, we get a callback with three values for `inputProps`, `fieldState`, and `formState`.

`inputProps` are important because you need to spread those over the `input` element. `inputProps` contains event handlers and other important information for our `input` element. Then as you can imagine, `fieldState` contains various state details about the field and `formState` has details about the overall form. State is documented later in this file.

Also notice that through the `...rest` object, we are still passing the required `name` to `Field`

## Field Wrap Inputs

The above example allows us to have a field wrap, but the result so far is always an `input` element. What if we want other things like `textarea` or `select`?

We can use a similar strategy that `Field` uses when it's not being wrapped - that is to pass a component as a prop into `FieldWrap`:

```jsx
const Input = props => {
  const { name, type, ...rest} = props
  return <input type={type || 'text'} id={`field-` + name} name={name} {...rest} />
}

const FieldWrap = props => {
  const { label, type, name, component: Component, ...rest } = props

  return (
    <Field {...rest}>
      {(inputProps, fieldState, formState) => (
        <div className="field-wrap">
          <label htmlFor={`field-` + name}>{label}</label>
          <div className="input">
            <Component {...inputProps} name={name} type={type} />
          </div>
          <div className="error">
            {fieldState.error}
          </div>
        </div>
      )}
    </Field>
  )
}

const LoginForm = props => (
  <Form>
    <FieldWrap label="Email" name="email" component={Input} />
    <FieldWrap label="Password" name="password" type="password" component={Input} />
  </Form>
)
```

With `Input` as a component and now abstracted away from `FieldWrap`, you can probably imagine making similar components for `Select`, `Textarea`, and more.


## Field Abstractions

Further abstraction can be done by making specific types of fields for quick use. For example, imaging having a `<FieldFirstName />` component which provides the same result as `<FieldWrap label="First Name" name="firstName" component={Input} />`.

With our new `FieldWrap` component, we can now make easy field abstractions for common fields:

```jsx
const FieldEmail = props => <FieldWrap label="Email" name="email" component={Input} {...props} />
const FieldFirstName = props => <FieldWrap label="First Name" name="firstName" component={Input} {...props} />
const FieldLastName = props => <FieldWrap label="Last Name" name="lastName" component={Input} {...props} />
```

To be used like this:

```jsx
const SignupForm = props => (
  <Form>
    <FieldEmail />
    <FieldEmail label="Repeat Email" name="repeatEmail" />
    <FieldFirstName />
    <FieldLastName />
  </Form>
)
```

We can even override defaults by still passing in props for `label` and `name`.


## Validation

To provide custom validation, pass a `validate` callback into `Form`. The `validate` callback gets called with every value change of any field. It receives the form's values as an argument and is expected to return an object of errors with the name of the field as the respective property of the return object:

```jsx
class LoginForm extends React.Component {
  validate(values) {
    const errors = {}
    if (!/^[\w\d\.]+@[\w\d]+\.[\w]{2,9}$/.test(values.email)) errors.email = "Invalid Email"
    if (!/^[\w\d]{6,20}$/.test(values.password)) errors.password = "Invalid Password"
    return errors
  }

  render() {
    return (
      <Form validate={this.validate}>
        <FieldEmail />
        <FieldPassword />
      </Form>
    )
  }
}
```

There are much more elegant ways of writing validation such that you wouldn't have to re-write rules for every form, but this shows the basic point that if the `email` and `password` fields are not valid, then we will return:

```json
{ "email": "Invalid Email", "password": "Invalid Password" }
```

See `fieldState` below to see how we can customize the error response to our liking.

## Submit Handling

To provide custom submit handling, pass an `onSubmit` callback into `Form`. The `onSubmit` callback gets called when the form is submitted and passes the form's values and `formState` into the callback.

```jsx
class LoginForm extends React.Component {
  validate(values) {
    const errors = {}
    if (!/^[\w\d\.]+@[\w\d]+\.[\w]{2,9}$/.test(values.email)) errors.email = "Invalid Email"
    if (!/^[\w\d]{6,20}$/.test(values.password)) errors.password = "Invalid Password"
    return errors
  }

  onSubmit(values, formState) {
    return axios.post('some/path', values)
  }

  render() {
    return (
      <Form validate={this.validate} onSubmit={this.onSubmit}>
        <FieldEmail />
        <FieldPassword />
        <button type="submit">Submit</button>
      </Form>
    )
  }
}
```

Note that your submit handler callback will not be called if the form is invalid based on your validation rules.

`onSubmit` is expected to return a promise. In this case we're using the XHR promise library [axios](https://github.com/mzabriskie/axios), but you can do anything you want with the values, as long as a promise is returned.

When a rejected promise is returned to the API, the `formState.submitFailed` flag is turned to `false`. To see more about how state is handled in the form, see `formState` below.


## Initial Values

The `Form` component can also take a prop for `initialValues`. This is a simple object with keys that match the names of the fields:

```jsx
const initialValues = { email: 'example@example.com', password: 'abc123' }
<Form initialValues={initialValues}></Form>
```


## Accessing `formState`

Sometimes it's nice to know what the `formState` is at the top-level of the API. For instance, what if we want to disable the submit button based on whether the form is valid?

The `Form` component allows us to pass a callback function as its first child:

```jsx
class LoginForm extends React.Component {

  validate() { return {} }
  onSubmit() { return Promise.resolve() }

  render() {
    return (
      <Form validate={this.validate} onSubmit={this.onSubmit}>
        {(props, formState) => (

          <form {...props}>
            <FieldEmail />
            <FieldPassword />
            <button type="submit" disabled={!formState.validForm || formState.submitting}>Submit</button>
          </form>

        )}
      </Form>
    )
  }
}
```

The only catch is that we now need to return a `<form>` element since using the API this way won't provide it for us. The callback though does provide `props` which we can give the `form` for it's callbacks that were provided to `Form`. The callback also provides `formState` which is the primary reason to use this pattern.


## `fieldState`

_todo_


## `formState`

_todo_
