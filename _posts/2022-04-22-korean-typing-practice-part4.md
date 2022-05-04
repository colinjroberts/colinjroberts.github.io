---
layout: post
title:  "Korean Typing Practice Part 4"
date:   2022-04-22 19:10:00 -0800
categories: programming
published: true
lede: "Part 4 of making a browser interface for a web tool to improve my ability to type in Korean. This post focuses on the logic of checking user input against generated sentences"
---

This post is part of a series documenting the processes of building a web app to practice typing in Korean.
- Part 1: [Generating random "words" from a given input]({% post_url 2022-02-28-korean-typing-practice-part1 %})
- Part 2: [Creating a visual English/Korean keyboard that responds to clicks and key presses]({% post_url 2022-03-07-korean-typing-practice-part2 %})
- Part 3: [Porting sentence generation code from Python to JavaScript and testing it]({% post_url 2022-03-21-korean-typing-practice-part3 %})
- This post: Handling text as keys are pressed: Hangul vs Latin alphabet edition

At the end of the last post, I had working random "word" generation in English and Korean, now written in JavaScript! My next step for getting a working MVP (one that I can used to practice typing in Korean) is to write the logic that compares my keyboard input to the generated "words" on screen.


## The Spec
Generally speaking, once words are generated, I want to type on my keyboard and know whether I've typed the expected letter or not. The experience will work like this:
- Generate text on screen. The letters that make up these words will be referred to as "expected letters" with the first expected letter being the first letter of the first word.
- While there are letters still expected, listen for key presses.
- Type a key:
    - If the typed letter is the expected letter, change the color from black to grey and expect the next letter in the word
    - If it is not the correct letter, change the color of the expected letter to red. Subsequent incorrect key presses have no change.
    - Typing backspace should move the expected letter back by one.

This approach is pretty straight forward for English where words are made up of letters. In Korean, words are made of syllable blocks, each of which is a unique unicode character. When a key is pressed, one Korean letter (*jamo*) is produced. Most modern interfaces I've used handle Korean input by making a sort of temporary syllable block builder that shows you the syllable block you have written so far, but doesn't commit the unicode character for that syllable block until you move on to the next syllable (either by pressing space or when there are no more possible syllables that can be made by adding jamo). This leads to a problem: How ho we compare each user key press if the characters aren't registered until after several keys are pressed?

The most direct corollary to the code for English would be to compare each syllable block after it is written. In that situation, even if I had a visual showing the characters that were forming the syllable as you type them, the user doesn't get real feedback until typing 2-4 letters. Since the point of this tool is to get direct feedback on whether or not I'm typing the correct key, this doesn't seem like the best approach.

What I really need is a way for each key press to highlight the specific character in each word to show me whether or not I've typed the right one. This will be an interesting change! As it stands, the program can handle 3 forms of Korean (syllables which decomposed, the various forms of composable jamo that might be accidentally pasted, and independent/dictionary form jamo) as input and uses them to generates precomposed syllable blocks for all of the text. 

{% 
include caption-img.html 
image="korean-typing-practice-part4-1.gif" 
caption="The shapes of individual jamo change depending on where they are in the syllable" 
alt = "Korean letters are typed one by one and change shape slightly as they form syllable blocks"
%}

<!-- Perhaps there is another way? Unicode also has a standard for handling Korean input as a series of composable jamo such that each jamo that makes up syllables could be written one after the other, and it is up to the system displaying the Korean words to put them together in syllables visually. [This pdf from Unicode](http://www.unicode.org/versions/Unicode1.1.0/appC.pdf) in section C.3.2 notes that for a composing system to work well, each jamo might need multiple forms (i.e. all Korean syllables fit into a rectangle, and depending on which jamo are in the syllable, sometimes a jamo will need to be taller or shorter or wider etc.). At this point in my understanding, I think that OSs and programs handling Hangul might have some discretion in how it is handled. My experience has been that as one types a character, the letters shift to the shape that is appropriate for the characters available:-->

I would rather have each sub-shape of the character change color. HOW COOL WOULD THAT BE!? So the question is now can I generate the non-standard, fully decomposed version of each letter AND selectively color it?  I think that would just be so cool, so how can it be done?

I first needed to refresh my memory (and learn a little more) about how Unicode handles Korean. [This github issue](https://github.com/MicrosoftDocs/typography-issues/issues/587) was an interesting look into how Korean implementation evolved at Microsoft, and [this technical report from unicode](http://unicode.org/L2/L2009/09052-tr47.html) was interesting but a tad hard to follow, and maybe incomplete. 

A little experimenting in FireFox suggests that I couldn't use the Unicode decomposed form (NFD) in which I use the composable forms wrapped in their own span and change the color. The color of the first character listed becomes the color of the whole thing:

---
<div>
<span style="color: red;">ᄀ</span>
<span style="color: green;">ᅡ</span>
<span style="color: blue;">ᆫ</span>
<span style="color: red;">ᄀ</span><span style="color: green;">ᅡ</span><span style="color: blue;">ᆫ</span>
</div>
---

```
<span style="color: red;">ᄀ</span>
<span style="color: green;">ᅡ</span>
<span style="color: blue;">ᆫ</span>
<span style="color: red;">ᄀ</span><span style="color: green;">ᅡ</span><span style="color: blue;">ᆫ</span>

```

---
<div>
<span style="color: red;">a</span>
<span style="color: green;">b</span>
<span style="color: blue;">c</span>
<span style="color: red;">a</span><span style="color: green;">b</span><span style="color: blue;">c</span>
</div>
---

```
<span style="color: red;">a</span>
<span style="color: green;">b</span>
<span style="color: blue;">c</span>
<span style="color: red;">a</span><span style="color: green;">b</span><span style="color: blue;">c</span>

```


To me, this suggests that the browser is converting those decomposed pieces into the syllable Unicode character. Perhaps this is not a surprise. It makes sense from a typography perspective to convert syllables to well design unicode characters as opposed to some kind of on the fly character reshaping and rendering.


After a little more searching, I don't see a clear and quick path forward to doing what I want directly. If I really deeply wanted the components of a Korean syllable to change color as I typed, I think the next step would be to write a tool to generate a "font" of SVGs in which each syllable is a group of addressable SVGs. I could render the SVGs on the web page and iterate across sibling marks as needed to change the color. This is one large yak that needs shaving and aside from this project, being able to generate a single stroke SVG Korean font would be helpful for using the pen plotter! Despite how cool of a project that would be, I have an MVP to make, so I will set that idea aside for another day.

Instead, I will adjust my desired functionality. Imagine again that the Korean text to practice typing has just been generated. The syllable I am meant to type will have a different background color than the others and above the main text, I will display as individual jamo characters the 2-4 jamo that make up the syllable. Those jamo will change color as I type, turning gray as I type the correct characters. When I finish the syllable, the whole syllable will turn gray, and the highlight moves to the next syllable, and the jamo for that are displayed in the area above. 


To make this happen, I'll add and change a few things:
- Add a div element in the HTML that I can use to insert the jamo if Korean is being used
- Change the generating function such that instead of only returning a string, it also returns an array of characters
- Change the code for inserting the string of generated text such that each character is inserted as a span with its own styling that can later be changed
- Add a way to keep track of which character is expected for the person to type
- Add a function tha verifies key presses and increments the character counter
- For Korean, add a function that breaks each syllable character into its jamo and displays those above the main typing area
- Add a function for handling deletes and doing the above process in reverse


## Prepping the HTML

At this stage, the boy of that app's html consists of (from the top down) an area for words that should be typed `word-typing-area`, a keyboard of letters that change color when typing and can be clicked to turn those letters on and off for word generation `keyboard`, and a form for inputting options `letter-selection`.

To allow fo Korean syllable decomposition, I divided the `word-typing-area` into two sections: `pre-jamo-expansion-area` and `word-display`.  

The full HTML now looks like this:
```html
<html>
    <head>
    <-- link to stylesheet omitted -->
    </head>
    <-- link to js script omitted -->

    <body class="grid-container">
        <div class="main">

            <div class="word-typing-area" id="word-typing-area">
                <div id="pre-jamo-expansion-area"></div>
                <div class="word-display" id="word-display"></div>
            </div>
            
            <div class="keyboard" id="keyboard">
                {{ keyboard|safe }}
            </div>

            <div class="letter-selection">
                <form id="letter-selection-form" action="/" method="GET">
                    <div class="form-group">
                        <label for="letters">Letters to practice:</label>
                        <input type="text" id="letters" name="letters" required size="20">
                    </div>
                <div class="form-group">
                    <label for="count">Number of words:</label>
                    <input type="number" id="count" name="count" required size="4">
                </div>
                <div class="form-group">
                    <input type="radio" id="langChoice1"
                            name="lang" value="EN">
                    <label for="langChoice1">English</label>
                    <input type="radio" id="langChoice2"
                            name="lang" value="KO">
                    <label for="langChoice2">Korean</label>
                    </div>
                <button type="submit">Update Words</button>
                </form>
            </div>
        </div>
    </body>
</html>
```

## Rendering Characters With Style
To be able to style the text that is on the screen, I changed how the getText() function returns its generated text. Instead of just returning a string, it now returns an object with two properties: `text` a space separated string of the generated words, and `decomposedText` an array of the characters used in each word. In English, the array will have the same characters as the string, but for Korean the string will have syllable characters whereas the array will have the jamo that each character is made of (e.g. the string `"안녕"` with have the array `['ㅇ','ㅏ','ㄴ','ㄴ','ㅕ','ㅇ']`) The function now looks like this:

```javascript
function getText(countOfWordsToReturn, lettersProvided, language){
    
    let maxWordLength = 6
    let ACCEPTED_LANGUAGES = new Set(['EN', 'KO'])

    // ... input checking code hidden ... // 

    // Return only a space if only whitespace is provided
    if (! lettersProvided.trim()){
        return {
            text: ' ',
            decomposedText: []
        }
    }

    let generatedText = []
    let decomposedText = undefined

    // Return words made by corresponding langauge handler
    if (language == 'EN'){
        // generate random words using provided Englsih letters
        generatedText = handleEn(countOfWordsToReturn, lettersProvided, maxWordLength)
        // break the generated words down into an array of characters
        decomposedText = decomposeWordsEn(generatedText.join(""))

    } else if (language == 'KO'){
        // generate random words using provided Korean letters
        generatedText = handleKo(countOfWordsToReturn, lettersProvided, maxWordLength)

        // deconstruct generated words into an array or arrays of jamo for each character
        let arrayOfJamoArrays = decomposeWordsKo(filterAndSortKoreanLetters(generatedText.join("")))
        let merged = [].concat.apply([], arrayOfJamoArrays)
        decomposedText = merged

    } else {
        return null
    }

    return {
        text: generatedText.join(" "),
        decomposedText: decomposedText
    }
}
```

This generated text is used in the eventListener attached to the form's submit button such that when the form is submitted, words are generated then inserted into the page. Once the text has been generated, the string is split and inserted into the HTML document with each character getting its own `<span>` so it can be styled independently. That way, characters that are yet to be typed can be one color, the character that should be typed next can be highlighted, and the characters that have already been typed can be a different color. 

To keep track of which character is the current one, I needed to add a counter somewhere. I decided to keep that information as a data attribute to the `word-typing-area`, the container for both the `word-display` div which holds English or Korean words to be typed, and the `pre-jamo-expansion-area` which breaks down Korean words.

Finally, the words are inserted into the `word-display`.  If Korean, the `pre-jamo-expansion-area` is converted into `jamo-expansion-area` and filled with the jamo of the first syllable. If English, the `jamo-expansion-area`  is cleared if it exists, the English text fills the `word-display` div. All of these are inserted with the correct style, either unhighlighted text or highlighted maintext or jamo


```javascript
// Event listener for using form submit to change words on page
const letterSelectionForm = document.getElementById('letter-selection-form');
letterSelectionForm.addEventListener('submit', async (e) => {
    e.preventDefault();
    
    const formData = new FormData(letterSelectionForm).entries()
    const formFeq = Object.fromEntries(formData)

    // generate text as a string and array of characters
    let result = getText(formFeq.count, formFeq.letters, formFeq.lang)

    // Set and clear wordDisplay environment
    let wordDisplay = document.getElementById('word-display')
    let wta = document.getElementById('word-typing-area')
    wordDisplay.textContent = undefined

    // Fill wordDisplay with generated text
    for (let letter of result.text.split("")){
        let syllable = document.createElement("span")
        syllable.className = "unhighlighted-maintext"
        syllable.textContent = letter
        wordDisplay.appendChild(syllable)
    }
    
    // Clear word counter and highlight first word
    wta.dataset.currentCharacterIndex = 0
    wordDisplay.childNodes[wta.dataset.currentCharacterIndex].className = "highlighted-maintext"

    // Prep typing area for English
    if (formFeq.lang == "EN"){
        wta.dataset.language = "EN"
        // Check for jamo area and remove if it is there
        let jea = document.getElementById("jamo-expansion-area")
        if (jea != undefined){
            jea.textContent = undefined
            jea.id = "pre-jamo-expansion-area"
            jea.className = "pre-jamo-expansion-area"

        }

    // Prep typing area for Korean
    } else if (formFeq.lang == "KO"){
        wta.dataset.language = "KO"
        let jea = document.getElementById("pre-jamo-expansion-area")
        if (jea != undefined){
            jea.className = "jamo-expansion-area"
            jea.id = "jamo-expansion-area"
        }

        // Trigger jamo display
        insertJamoFromCharacter()
    }
})


function insertJamoFromCharacter(){
    let wordDisplay = document.getElementById('word-display')
    let wta = document.getElementById('word-typing-area')

    // deconstruct current word
    const nextCharacterToType = wordDisplay.childNodes[wta.dataset.currentCharacterIndex].textContent
    let jamoForTyping
    if (nextCharacterToType == " "){
        jamoForTyping = [[" "]]
    } else {
        jamoForTyping = decomposeWordsKo(filterAndSortKoreanLetters(nextCharacterToType))
    }

    // Set jamo expander
    const jea = document.getElementById('jamo-expansion-area')
    jea.textContent = undefined
    for (let jamo of jamoForTyping[0]){
        let spanForJamo = document.createElement("span")
        spanForJamo.className = "unhighlighted-expansiontext"
        spanForJamo.textContent = jamo
        jea.appendChild(spanForJamo)
    }
    wta.dataset.currentJamoIndex = 0
    jea.childNodes[wta.dataset.currentJamoIndex].className = "highlighted-expansiontext"
}
```

## Changing Style as You Type
As the user types, the highlighter should move across the text. Typed letters should become gray, and in the case of Korean, new jamo should be displayed as needed. To accomplish this, I added a whole set of mechanisms to the releaseKey event listener. 

Previously, pressing and releasing a key changed the color of the svg key element on the screen. Now I wanted to add the core logic of a typing trainer: whether or not one pressed the correct key! It needs to do the following:
- If the correct key is pressed:
    - Turn that letter grey, increment the counter, move the highlighter to the next letter
- If the incorrect key is pressed:
    - Do nothing for now. Maybe in the future, it can turn the letter red as a visual flag
- If backspace is pressed:
    - Turn the current letter black, decrement the counter, move the highlighter to the previous letter

For Korean, there is a little extra logic which is that instead of always incrementing to the next syllable character in the word, it needs to increment the jamo until the final jamo is reached, and only then move on to the next word. Also note that for Korean, I handled deletes in the same way I'm used to seeing it on my computer. If I'm in the middle of typing a syllable, pressing backspace removes jamo by jamo. But after I've written a complete syllable, backspace erases the whole syllable.


```javascript
/* Handles visual class and app state changes on key release including
    * - moving on-screen highlighted text upon correct input or backspace
    * - changing color of text that has been correctly typed
    * - changing color of keys on the on-screen keyboard
    */  
function releaseKey(e){

    // handle backspace
    if (e.code == "Backspace"){
        getPrevLetterToType()
    }

    if (e.code in keyMap){
        let keyElement = document.getElementById(keyMap[e.code]);
        let keyClassName = keyElement.className.baseVal

        if (keyClassName == "key-active-pressed"){
            keyElement.className.baseVal = "key-active-unpressed";
        } else {
            keyElement.className.baseVal = "key-inactive-unpressed";
        }

        // handle correct key
        const keyCoverElement = document.getElementById(keyElement.id + "-cover");
        const keyMarkings = keyCoverElement.getAttribute("id-markings")
        const wta = document.getElementById('word-typing-area')

        // For English, move forward if key matches current main-text character
        if (wta.dataset.language=="EN") {
            const wordDisplay = document.getElementById('word-display')
            const currentCharacter = wordDisplay.childNodes[wta.dataset.currentCharacterIndex].textContent
            if (keyMarkings.match(currentCharacter.toUpperCase())){
                getNextLetterToType()
            }
        // For Korean, move forward if key matches current jamo character
        } else if (wta.dataset.language=="KO"){
            const jea = document.getElementById('jamo-expansion-area')
            const currentJamo = jea.childNodes[wta.dataset.currentJamoIndex].textContent
            if (keyMarkings.match(currentJamo)){
                getNextLetterToType()
            }
        }
    }
}

/* Gets the next English letter or Korean jamo to be expected.
* Handles incrementing word-display and jamo-expansion-area.
*/
function getNextLetterToType(){
    const wordDisplay = document.getElementById('word-display')
    const wta = document.getElementById('word-typing-area')
    const language = wta.dataset.language

    if (language == "EN"){
        if (wta.dataset.currentCharacterIndex >= wordDisplay.children.length-1) {
            // Must be at last character, do nothing for now.
            window.alert("Congrats! You did it!")
        } else {
            // Move to next character
            updateCharacter(wordDisplay, wta, 1)
        }

    } else if (language == "KO"){
        const jea = document.getElementById('jamo-expansion-area')
        if(wta.dataset.currentJamoIndex >= jea.children.length-1){
            if (wta.dataset.currentCharacterIndex >= wordDisplay.children.length-1) {
                // Must be at last character, do nothing for now.
                alert("Congrats! You did it!")
            } else {
                // Move to next character
                updateCharacter(wordDisplay, wta, 1)

                // Display new jamo
                jea.childNodes[wta.dataset.currentJamoIndex].className = "unhighlighted-expansiontext";
                insertJamoFromCharacter()
            }
        } else {
            // Highlight next jamo
            updateJamo(jea, wta, 1)
        }
    }
}

/* Gets the previous English letter or Korean jamo to be expected.
* Handles incrementing word-display and jamo-expansion-area.
*/
function getPrevLetterToType(){
    const wordDisplay = document.getElementById('word-display')
    const wta = document.getElementById('word-typing-area')
    const language = wta.dataset.language

    if (language == "EN"){
        if (wta.dataset.currentCharacterIndex == 0) {
            // Must be at first character, do nothing for now.
        } else {
            // Move to previous character
            updateCharacter(wordDisplay, wta, -1)
        }

    } else if (language == "KO"){
        const jea = document.getElementById('jamo-expansion-area')
        if(wta.dataset.currentJamoIndex == 0){
            if (wta.dataset.currentCharacterIndex == 0) {
                // Must be at first character, do nothing for now.
            } else {
                // Move to previous character
                updateCharacter(wordDisplay, wta, -1)

                // Display new jamo
                jea.childNodes[wta.dataset.currentJamoIndex].className = "unhighlighted-expansiontext";
                insertJamoFromCharacter()
            }
        } else {
            // Highlight previous jamo
            updateJamo(jea, wta, -1)
        }
    }
}


/* Changes classes and counter for Jamo in jamo-expansion-area
*/
function updateJamo(jea, wta, incrementor){
    if (incrementor > 0){
        jea.childNodes[wta.dataset.currentJamoIndex].className = "typed-expansiontext";
    } else {
        jea.childNodes[wta.dataset.currentJamoIndex].className = "untyped-expansiontext";
    }
    wta.dataset.currentJamoIndex = parseInt(wta.dataset.currentJamoIndex) + incrementor
    jea.childNodes[wta.dataset.currentJamoIndex].className = "highlighted-expansiontext";
}

/* Changes classes and counter for characters in word-display
*/
function updateCharacter(wordDisplay, wta, incrementor){
    if (incrementor > 0){
        wordDisplay.childNodes[wta.dataset.currentCharacterIndex].className = "typed-maintext";
    } else {
        wordDisplay.childNodes[wta.dataset.currentCharacterIndex].className = "untyped-maintext";
    }
    wta.dataset.currentCharacterIndex = parseInt(wta.dataset.currentCharacterIndex) + incrementor
    wordDisplay.childNodes[wta.dataset.currentCharacterIndex].className = "highlighted-maintext";
}
```



{% 
include caption-img.html 
image="korean-typing-practice-part4-2.gif" 
caption="Look at that typing and deleting!" 
alt = "An on screen keyboard with a web form under it and empty space above it. A mouse cursor clicks on two of the keys, then fills in the short web form selecting Korean, choosing to generate one word, then submits. A nonsensical word made of 6 Korean syllable blocks appears above the keyboard. Keys on the screen light up turning blue when they are typed by the user. When the correct keys are typed, the words on the screen turn gray."
%}


See the code at [this commit](https://github.com/colinjroberts/korean-typing-practice/tree/732de9f8d1c4a0c8c629493dcdc8d0998230f942)



## Next Steps
The next steps are still to make:
- metrics to keep track of how well I type and which letters I should work on
- some kind of lesson structure / progression (and as part of that changing how random words are generated)
- post MVP: converting from svg elements to html buttons, revamping style

