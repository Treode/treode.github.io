
# Jekyll cannot automatically generate the layouts from Jade files. There is a plugin to build
# regular pages from Jade, but not layout templates. Instead we use this a good old-fashion
# Makefile to generate the HTML layout templates from the Jade templates; then we check both into.

# By default we build layout templates suitable for development work. Notably, these leave out
# the analytics code, so that you can edit and re-edit the site, and refresh and refresh it again
# in your browser, all without the refreshes counting in our site metrics.
dev:
	jade --out . --obj '{dev: true}' jade/*.jade

# This builds the templates for distribution, that is for pushing to github.
dist:
	jade --out . jade/*.jade

# This sets up jade in watch mode, so that you can edit the template and have it regenerated
# immediately. Using this in conjunction with Jekyll's watch mode gives you an experience that's
# not quite entirely unlike WYSIWYG editing.
watch:
	jade --out . --obj '{dev: true}' --watch jade/*.jade
