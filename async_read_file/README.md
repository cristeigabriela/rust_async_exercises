# async_read_file
Implements the function `async_read_file`, which takes in a `path`, and returns *something* that fulfills the requirement of implementing the trait `Future` with the `Output = Result<Vec<u8>, tokio::io::Error>`. Inside, it takes care of obtaining a `HANDLE` to the file at `path`, if any.

The trait is implemented by the `ReadFileApi` which is responsible for maintaining the state that will be referenced (and potentially modified) each `poll`. It does the following steps:

- Check if `self.handle.is_null() || self.handle == INVALID_HANDLE_VALUE`, basic error handling for the file.
- Checks if our buffer (`self.buf`) has enough space for the next call, having to fit in, at maximum, a buffer of size `POLL_READ`. We just `extend` `self.buf` with `[0; POLL_READ]`.
- We call to the WinAPI function `ReadFile`, we make sure to provide a properly offset buffer (using `self.bytes_read`) to have the read  done at the right point. Moreso, we store how many bytes were read into `bytes_read`.
- We check if `ReadFile` had a non-zero return (success), AND if the last call was responsible for reading **zero** bytes. If that's the case, it means that we finished reading the file (it was a success, AND there is no more bytes to read).
  - In that case, we truncate `self.buf` to `self.bytes_read`, and return that our `poll`-ing is over, with the buffer.
- If we didn't enter the following code block, we add the bytes read in this `poll` to `self.bytes_read`.
- We check the WinAPI function `GetLastError`. If it's not `ERROR_SUCCESS`, we return that our `poll`-ing is over, with a raw OS error.
- We wake the task.
- We return that the task is still pending. 
