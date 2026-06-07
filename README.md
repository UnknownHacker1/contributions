# CodePath Open-Source Contribution

## Issue
**Project:** [portainer/portainer](https://github.com/portainer/portainer)
**Issue:** [#13199 – Missing error notification toast on deleting a used image](https://github.com/portainer/portainer/issues/13199)

## Why I Chose This Issue

I picked issue #13199, "Missing error notification toast on deleting a used image,"
in portainer/portainer. It's a small, contained frontend bug in a TypeScript and
React codebase, which lines up well with what I already do. I build React
frontends, so working with component state and notification patterns feels
familiar to me.

A few reasons it stood out:

1. The scope is clear. It's about showing an API error that already exists, using
   the app's existing toast system. There's no new feature, no API change, and no
   design work needed.
2. It's a genuine user-experience problem. The backend already returns a 409
   Conflict with a helpful message, but right now the frontend quietly ignores it,
   so the user has no idea why the delete failed.
3. Portainer is a big, actively maintained project (37.7k stars), which makes it a
   great place to learn how a real-world React app is organized and how a team
   handles contributions.
4. Personally, I want to get better at jumping into a large unfamiliar codebase,
   following an action from the UI down to the API call, and reusing what's already
   there instead of reinventing it.

After reading through the issue, here's my understanding of the problem. When you
delete a Docker image that's currently in use, the backend responds correctly with
a 409 and an error message. The problem is on the frontend: the delete-image flow
never passes that message to the notification system, so no error toast shows up.
This used to work before version 2.33.7. My plan is to update the delete-image
error handling so it displays the API's message as an error toast, the same way
the app already reports other errors.

I've also left a comment on the issue introducing myself as a CodePath student and
asking to be assigned. I'll update this section once a maintainer replies.
