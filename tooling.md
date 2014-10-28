### Build & Deployment Process

Projects should always attempt to include some generic means by which source can be linted, tested and compressed in preparation for production use.
Depending on what you need to do, use Grunt, Gulp, or both. Grunt is a task runner that can run builds, where Gulp is a build runner which can run tasks.

#### Use Gulp when you're:
 * Transforming code/css/html (building)
 * Running a linter
 * Doing operations on every save (with grunt watch)
 * Caring more about simple piping syntax

#### Use Grunt when you're:
 * Launching Servers
 * Needing very specific tasks to be done
 * Livereloading (Gulp has this now too!)
