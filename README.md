# My Website Optimization Project
## Lesson 4 - Udacity FEND

The challenge at hand was to optimize certain portions of an existing website for different metrics.

1. Optimize the index.html page to achive a Page Speed Insights ranking of 90 or better on both mobile and desktop.
2. Optimize the slider widget on views/pizza.html to resize in < 5ms as indicated in the console.
3. Optimize the views/pizza.html to achieve 60FPS or better on the scroll event.

### Part 1: Optimize index.html

The first most obvious part here was to get the images down to a reasonable size! In particular, Cam's meme was enormous at 263K or so. However, all the images could be optimized to shrink their file sizes to dramatically decrease the load on the network. I was even able to shave a few K off the small, round profile pic without major impact. Initially, I used this [gulp npm](https://www.npmjs.com/package/gulp-image-optimization)

It did a good job, but, since I'm adept at Photoshop, I reopened them and found that could fine tune out several more K w/ Photoshop's tools. Obviously, this wouldn't be practical on a large project, but I felt it worth noting.

My next step was to optimize the css. First I added the media attribute to the print.css stylesheet and set its value to "print", of course, alerting the browser it doesn't need it for first render. Then I minified and inlined the necessary CSS into the head of the document; eliminating a request and a couple K of CSS.

On to the JS... I used jsUglify to minimize the code and added async attribute to the script tags to allow the page to load and scripts to run when ready.

While I read about gzip compression and setting longer expire times for HTTP headers, I did not implement them as I'd already, with the above optimizations, achived scores of 96 for both desktop and mobile in Page Speed Insights.

![page speed desktop](https://github.com/hellbertos/web-performance/blob/my-updates/img/psi-desktop.jpg)

![page speed mobile](https://github.com/hellbertos/web-performance/blob/my-updates/img/psi-mobile.jpg)

### Part 2: Optimize Slider Widget

The main optimization to begin with here, which was discussed in class, was to remove the added calculations and intracacy of the determineDx function since it wasn't necessary when the sizeSwitcher function could merely return a value to be used as a percentage. A much more elegant and significantly quicker solution!

```javascript
// Set cases to return the new percentage width based on the slider input
    function sizeSwitcher (size) {
      switch(size) {
        case "1":
          return 25;
        case "2":
          return 33.33;
        case "3":
          return 50;
        default:
          console.log("bug in sizeSwitcher");
      }
    }
    // Get the new size percentage and set it to var newsize
    var newsize = sizeSwitcher(size);
```

Next up was to move the selection of the pizza containers outside of the for loop, so the js engine didn't need to parse the document each time through the loop. I changed the selection method to getElementByClassName instead of querySelectorAll as there is a small boost in performance using that former method; as per ( [https://jsperf.com/getelementsbyclassname-vs-queryselectorall/18]). This is implemented throughout the code anywhere querySelectorAll is used.

An additional small optimization was to use the length method only once on the returned DOM element array and store it in a variable to use within the for loop. Again, the assumption being the js interpreter wouldn't need to look it up each time. This was actually not very perceptable when using Dev Tools. It seemed as though the bars dipped a bit lower, but extensive reading made me feel as though this was a helpful addition.

The relevant code:
```javascript
var allPizzas = document.getElementsByClassName("randomPizzaContainer");
    var allPizzasLength = allPizzas.length;

    // Iterate through the pre-fetched elements and batch reset their width to the new size
    for (var i = 0; i < allPizzasLength; i++) {
      allPizzas[i].style.width = newsize+'%';
    }
```
![Pizza Size Slider Shot](https://github.com/hellbertos/web-performance/blob/my-updates/img/pizza-slider-bigger.JPG)


### Part 3: The Dreaded Sliding Pizzas

The first step in the quest for 60FPS on scroll for these sliding pizzas of doom was reduce the number of pizzas being creapackted in the first place. The initial loop created 200 pizzas... 200 PIZZAS!! Considering there are 8 columns mandated at a height of 256px, the determination for the number of rows is ascertained by querying the height of the viewport, dividing it by 256 and adding 1 (to insure that the first row is always drawn). Multiplying rows and columns gives the optimum number of pizzas for creatation.

As we learned in class, it is important to query the state of the DOM outside of the loop then batch process as efficiently as possible, so it was time to look at the loop. The next obivous optimization was to move the phase calculation out of the loop since it had a layout triggering scrollTop within it. That could be queried once outside the loop. Two small additional improvements were to select elements using getElementById and to use the length method on it once and store in a var; as explained above.

Next, in what I thought was a very interesting approach (Full credit to Udacity Mentor mcs - that person helped me out a ton!), was to declare an array and store the phase calculation results in it, so that they could be referenced rather than continually calculated within the loop. When inspected, those numbers repeat, so it isn't necessary to continue to replicate them. Very cool and decent performance gain. Still well over 60FPS though.

The next optimization, and very large performance gain, was when I added "will-change: transform" to the pizza element's "mover" class declaration in the stylesheet. That gave an enormous boost to performance and pretty much put me mostly under 60FPS. As an additional optimization in concert with the preceeding, backface-visibility: hidden was added to boost performance in browsers which support hardware acceleration.

The last optimization I made was (at the suggestion of Udacity Mentor mcs) was to move the pizzas using transform rather than manipulating the left value. This was interesting because, initially, the pizzas were all moved several hundred pixels over and overlapping.

The relevant code:
```javascript
var items = document.getElementsByClassName('mover');
  var scrollPos = document.body.scrollTop;
  var cachedLength = items.length;

  /** Declare an array to store the phase values generated by the the for loop which
      follows. Push those numbers into the array to decrease load on the browser to continually
      look those values up.
      Credit to mcs, Udacity Mentor, from this post:
      https://discussions.udacity.com/t/project-4-how-do-i-optimize-the-background-pizzas-for-loop/36302
  **/
  var constArray = [];
  for (i = 0; i < 5; i++) {
      constArray.push(Math.sin((scrollPos / 1250) + i % 5));
  	}
  var winWidth = window.innerWidth;


  for (var i = 0; i < cachedLength; i++) {
    var phase = constArray[i % 5];
    items[i].style.transform = 'translateX('+ 100 * phase +'px)';
  }
```
Additionally, to get the proper start value for translateX work correctly, the code block which generates the pizzas needs the initial left is set with "style.left" vs the original approach of "elem.basicLeft".

Other enhancements include creating the var elem outside of the loop and selecting the movingPizzas1 div outside of the loop so they don't need to be continually looked up by the interpreter. Addtionally, the "movingPizzas1" element is selected with getElementById rather than the slower querySelector.
```javascript
    document.addEventListener('DOMContentLoaded', function() {
      var cols = 8;
      var s = 256;
      var viewportHeight = window.innerHeight;

      /**
          Get the viewport height to determine the number of rows necessary to display then multiply it
          by the number of columns to determine how many pizzas are needed to display, as per Udacity Reviewer suggestion
      **/
      var rowsNeeded = Math.floor(viewportHeight / 256) +  1;
      var pizzasNeeded = cols * rowsNeeded;

      /**
          Create the var elem outside of the for loop to prevent its continued creation each time through the loop; as per Udacity Reviewer
      **/
      var elem;
      var movingPizzasDiv = document.getElementById("movingPizzas1");
      // Reduce Number of generated pizzas to approximately the number actually shown on-screen
      for (var i = 0; i < pizzasNeeded; i++) {
            elem = document.createElement('img');
            elem.className = 'mover';
            elem.src = "images/pizza.png";
            elem.style.height = "100px";
            elem.style.width = "73.333px";
            elem.style.left = (i % cols) * s + 'px';
            elem.style.top = (Math.floor(i / cols) * s) + 'px';
            movingPizzasDiv.appendChild(elem);
          }
    }
```
![Sliding Pizza Timeline](https://github.com/hellbertos/web-performance/blob/my-updates/img/pizza-scroll-timeline-update.jpg)