---
inFeed: true
hasPage: true
inNav: false
inLanguage: null
starred: false
keywords: []
description: ''
datePublished: '2015-11-24T10:45:52.696Z'
dateModified: '2015-11-24T10:45:27.815Z'
title: Color Palette Documentation for Living Style Guides
author: []
authors: []
publisher:
  name: null
  domain: null
  url: null
  favicon: null
sourcePath: _posts/2015-11-24-color-palette-documentation-for-living-style-guides.md
published: true
url: color-palette-documentation-for-living-style-guides/index.html
_type: Article

---
# Color Palette Documentation for Living Style Guides

Virtually every style guide has a color palette section in its documentation. Many times I've seen this documentation created manually, where every change to a color requires updating the values in two places -- the stylesheet and the style guide. This often leads to one falling out of sync with the other, and makes maintaining a living style guide more difficult.

The problem with this approach is that the values are being defined in two different places. For a true living style guide, the code should serve as the single source of truth. Extracting color values from CSS can help keep documentation in sync, expose outdated colors, and point out opportunities for normalizing designs.

The following will show how to use Node.js to build source-code-derived documentation for color palettes and present color contrast values for each possible pairing as a guide for usage.

**Initial Setup**

If you're already using Node.js in your build system, you can integrate this directly into your project. For the purposes of this tutorial, create a new sandboxed project to experiment with. 
> 
> mkdir color-documentation

> cd color-documentation

> npm init

Install the following modules.

npm i --save-dev postcss color colorable loads

The PostCSS module will be used to transform the source CSS into an AST for manipulation with JavaScript. Color will be used to convert the values found in the stylesheet to hexidecimal. Colorable will be used to get color contrast values and WCAG scores for every possible pairing of colors. And lodash will be used to find unique color values and to render templates to HTML.

Copy a CSS file into the project to use as a source for the color values. This tutorial will use Basscss as an example.

**Build Script**

Create a new build.js file that will be used to read the CSS file and eventually to generate an HTML page.

// build.js
var fs = require('fs')

// Read the contents of basscss.css

var css = fs.readFileSync('basscss.css', 'utf8')

console.log(css)

For now, this script just logs the stylesheet.
Package Scripts
Add a script to package.json for the build script.
// package.json
"scripts": {
"start": "node build"
}
Run the script to make sure everything is working.
npm start
Parsing CSS
Create a new lib/parse-colors.js file for parsing the CSS.
// lib/parse-colors.js
var \_ = require('lodash')
var postcss = require('postcss')
var Color = require('color')
module.exports = function(css) {
// Array of colors to return
var colors = \[\]
// Parse the CSS file and get the AST
var root = postcss.parse(css)
// Iterate through each declaration and log the property
root.eachDecl(function(decl) {
console.log(decl.prop)
})
return colors
}
Use this module in build.js.
// build.js
var fs = require('fs')
var parseColors = require('./lib/parse-colors')
// Read the contents of basscss.css
var css = fs.readFileSync('basscss.css', 'utf8')
var colors = parseColors(css)
The parse-colors module takes raw CSS and transforms it into an AST that can be iterated over. For now, this just logs all the properties found in the stylesheet.
Filtering Colors
Next update the parse module to filter out everything but color and background-color properties, convert the values to hex, and remove duplicates. While color values can be used in other properties including the background shorthand, handling that would add complexity and will be omitted from this tutorial.
// lib/parse-colors.js
var \_ = require('lodash')
var postcss = require('postcss')
var Color = require('color')
module.exports = function(css) {
var colors = \[\]
var root = postcss.parse(css)
// Iterate through each color and background-color declaration
root.eachDecl(/(color|background\\-color)/, function(decl) {
// Add each color value to the colors array
colors.push(decl.value)
})
// Convert all color values to hex strings
colors = colors.map(function(color) {
var hex
try {
hex = Color(color).hexString()
return hex
} catch(e) {
// Handle color values like inherit and currentcolor
return false
}
})
.filter(function(value) {
// Remove false values
return value
})
// Remove duplicate values
colors = \_.uniq(colors)
return colors
}
The first argument in root.eachDecl is a regular expression to filter declarations for either color or background-color. See the PostCSS documentation for more details. The Color().hexString() method converts any valid color value to hex format. The lodash \_.uniq method removes duplicate values from an array.
HTML Template
Next, create a template.html file for displaying the colors. This tutorial uses lodash templates, but any JavaScript templating language would work. Note: if you're using a stylesheet other than Basscss, the classes applied below may differ.
null

  * the .map() method is used to iterate over the colors array and passes the renderColor function as a callback. The renderColor function then renders an 
* for each color with a div that has its background color set to that value.
Edit build.js to read the template and generate an HTML file.
// build.js
var \_ = require('lodash')
var fs = require('fs')
var parseColors = require('./lib/parse-colors')
var css = fs.readFileSync('basscss.css', 'utf8')
// Read the template string
var template = fs.readFileSync('template.html', 'utf8')
// Create a lodash template function
var tpl = \_.template(template)
var colors = parseColors(css)
// Render the template function to an HTML string
var html = tpl({ colors: colors })
// Write an HTML file to disk
fs.writeFileSync('index.html', html)
Now run npm start which should generate an HTML file with the colors parsed from the stylesheet.
npm start && open index.html
Color palette demo
Readable Color Combinations
The Colorable module takes an array of colors and returns a nested array of color combinations, along with their contrast values to test for readability. This can be useful for seeing what foreground-background pairs can and can't be used for text.
Create a new lib/parse-combos.js file.
// lib/parse-combos.js
var colorable = require('colorable')
module.exports = function(colors) {
var combos = \[\]
// Get the Colorable array
var arr = colorable(colors)
// Flatten the array and combine the foreground and background colors
arr.forEach(function(color) {
var pairing = color.hex
color.combinations.forEach(function(combo) {
combo.pairing = pairing
combos.push(combo)
})
})
// Sort the array by contrast from high to low
combos.sort(function(a, b) {
return b.contrast - a.contrast
})
return combos
}
Add the parse-combos module to build.js and pass the combos array into the template function.
// build.js 
var \_ = require('lodash')
var fs = require('fs')
var parseColors = require('./lib/parse-colors')
var parseCombos = require('./lib/parse-combos')
var css = fs.readFileSync('basscss.css', 'utf8')
var template = fs.readFileSync('template.html', 'utf8')
var tpl = \_.template(template)
var colors = parseColors(css)
var combos = parseCombos(colors)
var html = tpl({
colors: colors,
combos: combos
})
fs.writeFileSync('index.html', html)
Add a section to template.html to display the color combinations.

## Combinations
* Aa