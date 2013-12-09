---
layout: post
title: "webapp2 sessions with the blobstore"
date: 2011-09-11 16:01
comments: true
categories: [app engine, blobstore, webapp2]
---

I recently started using [webapp2 sessions](http://webapp-improved.appspot.com/api/webapp2_extras/sessions.html), and was happy with how they work. However, when trying to access sessions during an upload to the blobstore, I was not able to since the BlobstoreUploadHandler did not inherit from the new BaseHandler I created as directed by the webapp2 session instructions. I wanted to do this to send a message to the user that their upload was successful, using the `add_flash()` function. Appending `?message=Upload successful.` to the redirect URL would have worked, but is lame.

Trying to use it through multiple inheritance also fails:
`class EntryUploadHandler(blobstore_handlers.BlobstoreUploadHandler, BaseUploadHandler):`
with some error.

The solution was to create a BaseUploadHandler class with special handling for sessions. It appears that the upload handler doesn't act the same as a normal RequestHandler, so you have to do everything in one place:
`class BaseUploadHandler(blobstore_handlers.BlobstoreUploadHandler):
	def add_message(self, level, message):
		store = sessions.get_store(request=self.request)
		session = store.get_session()
		session.add_flash(message, level, BaseHandler.MESSAGE_KEY)
		store.save_sessions(self.response)`

Full code [here](https://github.com/mjibson/journalr/blob/master/main.py).
