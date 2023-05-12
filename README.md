# The Problem

If you ever want to log when an error occurs and what caused it, you may find yourself using a `match` statement for every possible error instead of using the `?` operator.

This results in extremely verbose code. It's a pain to write and maintain.

# Introducing wrap-match!

wrap-match is an attribute macro that wraps your function in a `match` statement. Additionally, **it attaches rich error information to all statements using the `?` operator (aka try expressions).**
This allows you to know exactly what line and expression caused the error.

> **Note**
>
> wrap-match uses the `log` crate to log success and error messages. It does not expose the log crate for expanded functions to use; you must depend on it yourself.
>
> Additionally, **no messages will appear unless you use a logging implementation.** I recommend `env_logger`, but you can find a full list
> [here](https://docs.rs/log/#available-logging-implementations).

## Example

First, add this to your `Cargo.toml`:

```toml
[dependencies]
wrap-match = "1"
log = "*"
# You'll also want a logging implementation, for example `env_logger`
# More info here: https://docs.rs/log/#available-logging-implementations
```

Now you can use the `wrap_match` attribute macro:

```rs
#[wrap_match::wrap_match]
fn my_function() -> Result<(), CustomError> {
    Err(CustomError::Error)?; // notice the ?; when the macro is expanded, it will be modified to include line number and expression
    Ok(())
}
```

This would expand to something like this (comments are not included normally):

```rs
fn my_function() -> Result<(), CustomError> {
    struct _wrap_match_error<E> {
        line_and_expr: Option<(u32, String)>,
        inner: E,
    }

    // This allows you to return `Err(CustomError::Error.into())`
    impl<E> From<E> for _wrap_match_error<E> {
        fn from(inner: E) -> Self {
            Self { line_and_expr: None, inner }
        }
    }

    // This is where the original function is
    fn _wrap_match_inner_my_function() -> Result<(), _wrap_match_error<CustomError>> {
        Err(CustomError::Error)
            .map_err(|e| _wrap_match_error {
                // Here, line number and expression are added to the error
                line_and_expr: Some((3, "Err(CustomError::Error)".to_owned())),
                inner: e.into(),
            })?;
        Ok(())
    }

    match _wrap_match_inner_my_function() {
        Ok(r) => {
            ::log::info!("Successfully ran my_function");
            Ok(r)
        }
        Err(e) => {
            if let Some((_line, _expr)) = e.line_and_expr {
                ::log::error!("An error occurred when running my_function (when running `{_expr}` on line {_line}): {:?}", e.inner);
            } else {
                ::log::error!("An error occurred when running my_function: {:?}", e.inner);
            }
            Err(e.inner)
        }
    }
}
```

If we run this code, it would log this:

```log
[ERROR] An error occurred when running my_function (when running `Err(CustomError::Error)` on line 3): Error
```

As you can see, wrap-match makes error logging extremely easy while still logging information like what caused the error.

## Customization

wrap-match allows the user to customize success and error messages, as well as choosing whether or not to log anything on success.

### `success_message`

The message that's logged on success.

Available format specifiers:

-   `{function}`: The original function name.

Default value: `Successfully ran {function}`

Example:

```rs
#[wrap_match::wrap_match(success_message = "{function} ran successfully!! 🎉🎉")]
fn my_function() -> Result<(), CustomError> {
    Ok(())
}
```

This would log:

```log
[INFO] my_function ran successfully!! 🎉🎉
```

### `error_message`

The message that's logged on error, when line and expression info **is** available. Currently, this is only for try expressions (expressions with a `?` after them).

Available format specifiers:

-   `{function}`: The original function name.
-   `{line}`: The line the error occurred on.
-   `{expr}`: The expression that caused the error.
-   `{error}` or `{error:?}`: The error.

Default value: `` An error occurred when running {function} (caused by `{expr}` on line {line}): {error:?} ``

Example:

```rs
#[wrap_match::wrap_match(error_message = "oh no, {function} failed! `{expr}` on line {line} caused the error: {error:?}")]
fn my_function() -> Result<(), CustomError> {
    Err(CustomError::Error)?;
    Ok(())
}
```

This would log:

```log
[ERROR] oh no, my_function failed! `Err(CustomError::Error)` on line 3 caused the error: Error
```

### `error_message_without_info`

The message that's logged on error, when line and expression info **is not** available. This is usually triggered if you return an error yourself and use `.into()`.

Available format specifiers:

-   `{function}`: The original function name.
-   `{error}` or `{error:?}`: The error.

Default value: `An error occurred when running {function}: {error:?}`

Example:

```rs
#[wrap_match::wrap_match(error_message_without_info = "oh no, {function} failed with this error: {error:?}")]
fn my_function() -> Result<(), CustomError> {
    Err(CustomError::Error.into())
}
```

This would log:

```log
[ERROR] oh no, my_function failed with this error: Error
```

### `log_success`

If `false`, nothing will be logged on success.

Default value: `true`

Example:

```rs
#[wrap_match::wrap_match(log_success = false)]
fn my_function() -> Result<(), CustomError> {
    Ok(())
}
```

This would log nothing.

## Limitations

wrap-match currently has the following limitations:

1.  wrap-match cannot be used on functions in implementations that take a `self` parameter. If you need support for this, please create a GitHub issue with your use case.

1.  wrap-match only supports `Result`s. If you need support for `Option`s, please create a GitHub issue with your use case.

1.  `error_message` and `error_message_without_info` only support formatting `error` using the `Debug` or `Display` formatters. This is because of how we determine what formatting specifiers are used.
    If you need support for other formatting specifiers (such as `:#?`), please create a GitHub issue with your use case.

1.  wrap-match cannot be used on `const` functions. This is because the `log` crate cannot be used in `const` contexts.

If wrap-match doesn't work for something not on this list, please create a GitHub issue!