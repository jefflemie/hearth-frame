# hearth-frame

One HTML file. It puts a private Google Apps Script web app in an iframe,
which is the only way to suppress the blue "This application was created by a
Google Apps Script user" bar that Google serves above every one of them.

It has to be HOSTED rather than opened from disk. From `file://` the document
has the opaque origin `null`: the browser sends no Referer and treats its
storage as untrusted, so Google's `/exec` redirect cannot complete and it
answers as though nobody is signed in -- in Drive's voice, because an Apps
Script project is a Drive file, which is why the error never mentions sign-in.

**No address is in this repository.** The page asks for one on first open and
keeps it in that browser's `localStorage`. The app behind it is deployed
*execute as me, access only myself*, so anyone else opening this page gets a
Google sign-in wall and nothing else.
