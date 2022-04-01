---
layout: post
title:  "Korean Typing Practice Part 3"
date:   2022-03-21 19:10:00 -0800
categories: programming
published: true
lede: "Part 3 of making a browser interface for a web tool to improve my ability to type in Korean that focuses on porting Python to JavaScript and JS unit testing"
---

This post is part of a series documenting the processes of building a web app to practice typing in Korean.
- Part 1: [Generating random "words" from a given input]({% post_url 2022-02-28-korean-typing-practice-part1 %})
- Part 2: [Creating a visual English/Korean keyboard that responds to clicks and key presses]({% post_url 2022-03-07-korean-typing-practice-part2 %})
- This post: Porting Python to JavaScript and testing it

At the end of the last post, my next step for getting a working MVP ( one that I can used to practice typing in Korean) was adding logic to compare what I type to the generated words on the screen. By that, I mean I wanted words presented to me, then as I type letters on my keyboard, the letters I type would be compared to the ones on the screen so I know whether or not I'm typing correctly.

The question at hand is where the word generation code should live and where the typing/validation code should live. Server or client browser? 

I already know I want the part of the system that compares the user-typed letters to the generated words to be happening in the browser. Sending every keystroke to the server, responding, then changing browser state seems like it would unnecessarily introduce latency concerns and needing to manage connections. What about the "word" generation?

Word generation isn't very computationally heavy. The server certainly COULD do it, but what is gained? We'd know exactly what words were generated and could keep track if we wanted. Do we need to know that words were generated multiple times but never acted on? That sounds like a vector for abuse. Plus, if it were run on the client and we wanted to know what words were generated, we could send it over as needed. With that, I've convinced myself that the word generation should just happen in the browser, and data can be sent later if needed.

That means I've added a new step to take care of before I write new code for tracking keystrokes and comparing to the generated words. I now need to port the Python word generator code to JavaScript and make sure it works, which is what this post is about.


## Writing Tests
As I went through the process of porting the Python code to JavaScript by hand, I found myself wanting to unit test each method as I added it. I'll be using [Jest](https://jestjs.io/docs/getting-started) because I already have it installed. Jest's approach is very straightforward. Tests can be written on their own or in a bundle. To try to keep things a little organized, I made a subdirectory for scripts. By the end of my development, it looks like this: 
```
.
├── README.md
├── app.py
├── generate_text.py
├── keyboard.py
├── static
│   ├── scripts
│   │   ├── jest.config.js
│   │   ├── keyListeners.js
│   │   ├── wordDecomposer.js
│   │   └── wordDecomposer.test.js
│   └── style.css
└── templates
    └── app.html
```

With this structure, as along as I'm in that scripts directory, running `jest` at the command line runs my tests.

I like setting up my simple tests like this:
```javascript
// Note that either 'test' or 'it' can be used
test("human readable description of the test", () => {
    let expected = "This is the output!"
    let actual = methodIAmTesting()
    expect(actual).toEqual(expected)
})
```

Tests can be bundled using the `describe` method:
``` javascript
describe("methodIAMTesting", () => {
    // Note that either 'test' or 'it' can be used
    it("has a truthy output", () => {
        let expected = "This is the output!"
        let actual = methodIAmTesting()
        expect(actual).toBeTruthy()
    })

    it("correctly provides the output", () => {
        let expected = "This is the output!"
        let actual = methodIAmTesting()
        expect(actual).toEqual(expected)
    })
})
```

Here are a two tests from the top of the file:
```javascript
const wordDecomposer = require('./wordDecomposer');

describe("decomposeWordsEn with EN language flag", ()=> {
    test("decomposes (' ') into Set{}", () => {
        let output = new Set([])
        expect(wordDecomposer.decomposeWordsEn(' ')).toStrictEqual(output);
      });
    
    test("decomposes ('aabbcc') into Set{'a', 'b', 'c'}", () => {
        let output = new Set(['a', 'b', 'c'])
        expect(wordDecomposer.decomposeWordsEn('aabbcc')).toStrictEqual(output);
    });
    
    test("decomposes ('a2a-아b,b=c{c') into Set{'a', 'b', 'c'}", () => {
        let output = new Set(['a', 'b', 'c'])
        expect(wordDecomposer.decomposeWordsEn('a2a-아b,b=c{c')).toStrictEqual(output);
    });
    
    test("decomposes ('ᄀᄁㄴㄷ은') into Set{}", () => {
        let output = new Set()
        expect(wordDecomposer.decomposeWordsEn('ᄀᄁㄴㄷ은')).toStrictEqual(output);
    });
})

describe("getText Korean Integration with multiple words, multiple EN and KO letters and symbols", () => {
    let actualOutput = wordDecomposer.getText(4, 'a.bㅗ4c1d!ㄹ', 'KO')
    it("contains only the provided letters", () => {
        expect(actualOutput).toEqual(expect.stringMatching(/[로롤]/))
    })

    it("is of the correct length", () => {
        let numberOfWords = actualOutput.split(" ")
        expect(numberOfWords.length).toEqual(4)
    })
})
```
view the whole test suite as a [GitHub gist](https://gist.github.com/colinjroberts/9aeae715a510627ea49426e0efe24fb6)

Jests's output is pretty nice. For those two tests, it looks like this (but with color!):
```
> jest

 PASS  scripts/wordDecomposer.test.js
  decomposeWordsEn with EN language flag
    ✓ decomposes (' ') into Set{} (2 ms)
    ✓ decomposes ('aabbcc') into Set{'a', 'b', 'c'} (1 ms)
    ✓ decomposes ('a2a-아b,b=c{c') into Set{'a', 'b', 'c'}
    ✓ decomposes ('ᄀᄁㄴㄷ은') into Set{} (3 ms)
  getText Korean Integration with multiple words, multiple EN and KO letters and symbols
    ✓ contains only the provided letters
    ✓ is of the correct length (1 ms)

Test Suites: 1 passed, 1 total
Tests:       6 passed, 6 total
Snapshots:   0 total
Time:        0.442 s, estimated 1 s
Ran all test suites.
```

As I mentioned above, I wrote my tests while I ported the Python code by hand to get a sense of what the process was like, and it was a process!


## Porting Python to JS by Hand

I started out thinking that it wouldn't be too much work. I know and practice Python, I know and practice JavaScript, but it turns out I don't practice translating. It reminds me of when I occasionally transcribe audio. I know how to listen and understand. I know how to read and write and understand. But I definitely haven't practiced typing things as I hear them, and it always takes longer than expected (maybe I should explore [a different input method](https://asetniop.com)).

There are a number of things that need to be changed to for form Python to JavaScript:
- function definitions `def function_name():` ==> `function functionName(){}`
    - change colons to brackets
    - change method name case
- variable definitions `code_lang = 'PY'` ==> `let codeLang = 'JS'`
    - add JS variable declarations
    - change variable name case convention
- logical operators `or` ==> `||`, `and` ==> `&&`
    - and chained operators like `0 < i < 10` if I had any
- explicit type conversions like `int('2')` ==> `parseInt('2')`
    - implicit conversion differences:
        - `'a' * 2 == 'aa'` =/=> `'a' * 3 == NaN` 
        - `'3' * 3 == '333'` =/=> `'3' * 3 == 9`
- Python's floor division operator `a//b` ==> `Math.floor(a/b)`
- commenting and docstring style `#` ==> `\\` and `"""` ==> `\* *\`
- dict constructors to Map constructors `my_dict = {key1: val1, key2: val2}` ==> `let myMap = new Map([[key1, val1],[key2, val2]])`
    - different constructor style (though now that I'm typing this, should have wrapped the dict in `Object.entries()`)
    - changing the variable names with 'dict' to 'map'
- dict bracket accessors `my_dict['key']` ==> `myMap.get('key')` 
- Python builtin functions like string `ascii_letter`, `chr`, and `ord`
    - I ended up polyfilling these to see if that was an effective technique. I think it was!
- Python object methods like `list.append()` ==> `Array.push()`
- multiple returns `a, b, c = method_with_three_returns()` ==> `{a, b, c} = {...MethodReturnsObjectWithThreeItems}`
- for loops `for x in range(n):...` ==> `for(let i=0; i<n;i++){...}`
- format strings `f'string with {variable}'` to ``` `string with ${variable}` ```
- `if x:...elif y:...else:` ==> `if(x){...}else if(y){...}else{...}`
- and of course adding semicolons (if you believe in them)

Though I knew it was going to be some amount of work, I was actually a bit surprised at just how much needed to be done. 

Here is a comparison of the first few methods.

**Python**
```python
from random import randint
from string import ascii_letters

def get_text(count_of_words_to_return, letters_provided, language):
    """Check input for errors and route to handler by language
    """
    
    MAX_WORD_LENGTH = 6
    ACCEPTED_LANGUAGES = ('EN', 'KO')

    # Raise errors if inputs are wrong
    if not (count_of_words_to_return and letters_provided and language):
        raise ValueError(f"int count expected, {count_of_words_to_return} was " 
                           "provided; string letters expected, " 
                           "{letters_provided} was provided, "
                            "string lang was expected, {language} was provided.")
    elif int(count_of_words_to_return) <= 0:
        raise ValueError(f"int count must be greater than 0: "
                          f"{count_of_words_to_return} was provided")
    elif language not in ACCEPTED_LANGUAGES:
        raise ValueError(f"Langauge must be one of {ACCEPTED_LANGUAGES}: "
                         f"{language} was provided")
    
    # Return only a space if only whitespace is provided
    if not letters_provided.strip():
        return " "
    
    # Return words made by corresponding langauge handler
    if language == 'EN':
        return handle_EN(count_of_words_to_return, letters_provided, MAX_WORD_LENGTH)
    elif language == 'KO':
        return handle_KO(count_of_words_to_return, letters_provided, MAX_WORD_LENGTH)
    else:
        return None
    
        
def handle_EN(count_of_words_to_return, letters_provided, max_word_length):
    # Deduplicate list of letters
    set_of_letters = deduplicate_letters_EN(letters_provided)

    # Throw an error if there are no ascii letters in the set
    if len(set_of_letters) <= 0:
        raise ValueError(f"string letters expected at least 1 ascii letter, \
                        {set_of_letters} was provided")
    
    # Create and return the number of requested words
    output = create_many_words_EN(set_of_letters, count_of_words_to_return, max_word_length)
    
    return " ".join(output)


def handle_KO(count_of_words_to_return, letters_provided, max_word_length):
    # Filter input to only Korean letters
    dict_of_filtered_input = filter_input_KO(letters_provided)

    # Throw an error if there are no Korean letters in the set
    list_of_dict_contents_is_not_empty = [len(l) > 0 for l in dict_of_filtered_input.values()]
    if not any(list_of_dict_contents_is_not_empty):
        raise ValueError(f"string letters expected at least 1 korean letter, "
                         f"{letters_provided} was provided.")

    # Create a deduplicated set of letters
    set_of_letters = deduplicate_letters_KO(dict_of_filtered_input)
    
    # Create and return the number of requested words
    output = create_many_words_KO(set_of_letters, count_of_words_to_return, max_word_length)
    
    return " ".join(output)
    
```


**JavaScript**
```javascript
// Check input for errors and route to handler by language
function getText(countOfWordsToReturn, lettersProvided, language){
    
    MaxWordLength = 6
    ACCEPTED_LANGUAGES = new Set(['EN', 'KO'])

    // Raise errors if inputs are wrong
    if (! countOfWordsToReturn && lettersProvided && language){
        throw `int count expected, ${countOfWordsToReturn} was ` 
                        `provided; string letters expected, `
                        `${lettersProvided} was provided, `
                        `string lang was expected, ${language} was provided.`
    } else if (parseInt(countOfWordsToReturn) <= 0){
        throw `int count must be greater than 0: ${countOfWordsToReturn} was provided`
    }
    else if (!ACCEPTED_LANGUAGES.has(language)){
        throw `Language must be one of ${ACCEPTED_LANGUAGES}: ${language} was provided`
    }

    // Return only a space if only whitespace is provided
    if (! lettersProvided.trim()){
        return " "
    }

    let output = []
    // Return words made by corresponding langauge handler
    if (language == 'EN'){
        output = handleEn(countOfWordsToReturn, lettersProvided, MaxWordLength)
    } else if (language == 'KO'){
        output = handleKo(countOfWordsToReturn, lettersProvided, MaxWordLength)
    } else {
        return None
    }
    return output.join(" ")
}

// Filter and deduplicate input, generate words using that
function handleEn(countOfWordsToReturn, lettersProvided, MaxWordLength){
    // Deduplicate list of letters
    SetOfLetters = decomposeWordsEn(lettersProvided)

    // Throw an error if there are no ascii letters in the set
    if (SetOfLetters.length <= 0){
        throw ValueError(`string letters expected at least 1 ascii letter, `
                        `${SetOfLetters} was provided`)
    }

    // Create and return the number of requested words
    listOfGeneratedWords = createManyWordsEn(SetOfLetters, countOfWordsToReturn, MaxWordLength)
    
    return listOfGeneratedWords
}

// Filter and decompose input, generate words using that
function handleKo(CountOfWordsToReturn, LettersProvided, MaxWordLength){

    // Filter input to only Korean letters
    let mapOfKoreanLettersByType = filterAndSortKoreanLetters(LettersProvided)

    // Throw an error if there are no Korean letters in any of the lists
    let allListsAreEmpty = true
    mapOfKoreanLettersByType.forEach( (value, key) => {
        allListsAreEmpty = allListsAreEmpty && value.length == 0
    })


    if(allListsAreEmpty){
        throw ValueError(`string letters expected at least 1 korean letter, `
                         `${LettersProvided} was provided.`)
    }
    
    // Create a deduplicated set of letters
    setOfLetters = decomposeWordsKo(mapOfKoreanLettersByType)
    
    // Create and return the number of requested words
    listOfGeneratedWords = createManyWordsKO(setOfLetters, CountOfWordsToReturn, MaxWordLength)
    
    return listOfGeneratedWords
}
```
View the whole thing as a [GitHub gist](https://gist.github.com/colinjroberts/f3a4ca2b3b5525e7f2be3b88a44ab81c)

There was so much finding and replacing, it feels like it could have at least partially been automated, so I looked around for some tools that might have been able to do it for me. Maybe I was looking in the wrong place (or at the wrong time, one site suggests that Python to JS compilers were all the rage 10 or so years ago...many of the GitHub repos I found for these tools stopped updating around then.)


## Porting Python to JS With Tools
I searched around to try to find a transpiling tool that could convert my Python 3.9 into any form of JavaScript. Here are a few of the tools that came up in a quick search:

- [JavaScripthon](https://github.com/metapensiero/metapensiero.pj) - transpiler that can convert some Python to ES6
- [RapydScript](https://github.com/atsepkov/RapydScript) - precompiler that allows writing JS more like Python
- [Transcrypt](http://www.transcrypt.org) - Subset of Python that can be compiled to JS
- [PScript](https://github.com/flexxui/pscript) - Transpiler for a subset of Python
- [Brython](https://github.com/brython-dev/brython) - A compiler meant to allow you to include or write python directly into web source code and have it run in browser


I was interested to see how each tool handled the conversion and ideally take the JS that was output and run it against my tests. I should have known what I was in for.


### JavaScripthon
Right away from the repo, I was a little unsure of how this would go. It says Python 3.5 is required, but I see a few updates from the past few months that suggest it supports basic Python 3.9. So I tried with that first. I got the same error mentioned in this [Github Issue](https://github.com/metapensiero/metapensiero.pj/issues/53), and followed some of the mentioned solutions like installing directly from the repo. I also tried using Python 3.8 and got a few different errors (I think related to fancy tuple unpacking in a for loop). Unfortunately, I never got it working, and I had other tools to try, so I didn't mess around very much.

Javascripthon looks like a really neat tool, and I hope Alberto is able to continue advancing it.


### RapydScript
This popped up in several searches, but doesn't really seem to fit my needs. More than the other tools, this one really seems to be more of its own language that is Python-like that compiles to JavaScript. I will say that the output in the examples looks really nice, and I do think it is an interesting idea to have smaller languages that compile to other ones. (Perhaps someone has written one for [SPL](https://shakespearelang.com/latest)?)

Had I known about such tools before I started writing, this might be a viable option, but I didn't end up trying it.


### Transcrypt
Transcrypt says it supports Python 3.7 and just to be sure I wasn't using anything fancy and forbidden, I made a copy of `generate_text.py`, added the following lines to make it runnable, and tried it out in a 3.7 environment. 

```python
if __name__ == "__main__":
    print(get_text(5, 'abcd', 'EN'))
    print(get_text(5, 'ㄹ호ㅓㅏ', 'KO'))
```
(note to self, probably should have written Python tests too...if only I had a Jest to pytest converter!)

The python code ran well in 3.7. I wasn't expecting problems, but I have grown fond of some of the more recent Python tools like using '=' in my fstrings which would have caused problems.

By this point in my search, I really wanted something to compile, so I was determined to make it work. After a lot of fiddling, I was able to get transcrypt working at least a little bit. The changes I made to get it working were removing my import from random and changing how I wrote range() functions. 

I didn't replace random with a custom function, I just made those always select 0. This wouldn't result in a functioning program, but that's fine. I also noticed that it was throwing silent errors for using `range(x)` instead of `range(0,x)`. I suspect this is just nuance of [transcrypyt's "fairly extensive subset of Python"](http://www.transcrypt.org/docs/html/what_why.html)

Now because of those changes alone, I can't run my tests against the generated content, but I was also surprised to see that none of my literals seem to be coming across the conversion. Anywhere there should be a string or number, there is just white space. 

This project looks quite cool and I suspect it works quite well if one uses it for its intended purpose using the exact subset it uses.


### PScript

This one worked much more like I was expecting, but the process was a little different. The setup experience was much better than the other tools so far. I was able to install directly to conda with `conda install pscript -c conda-forge` (though it did take a long time...I should really look into [Mamba](https://mamba.readthedocs.io/en/latest/index.html)), then to test it out, I imported it directly into my Python code file. with `from pscript import py2js`, and added a few lines at the bottom of the file to have it convert my Python functions:
```python
if __name__ == "__main__":
    print(py2js(get_text))    
    print(py2js(handle_EN))    
    print(py2js(handle_KO))
```

With that fairly simple start up, it worked! Given python code, it produced JS. It made polyfills for various Python builtins, handled integer division and `range(x)`, and was generally pretty readable. It didn't handle random that well which isn't TOO surprising for my recently lowered expectations, so this was really exciting! The code above produced the following JS (with a little editing):

```javascript
var _pyfunc_format = function (v, fmt) {  // nargs: 2
    fmt = fmt.toLowerCase();
    var s = String(v);
    if (fmt.indexOf('!r') >= 0) {
        try { s = JSON.stringify(v); } catch (e) { s = undefined; }
        if (typeof s === 'undefined') { s = v._IS_COMPONENT ? v.id : String(v); }
    }
    var fmt_type = '';
    if (fmt.slice(-1) == 'i' || fmt.slice(-1) == 'f' ||
        fmt.slice(-1) == 'e' || fmt.slice(-1) == 'g') {
            fmt_type = fmt[fmt.length-1]; fmt = fmt.slice(0, fmt.length-1);
    }
    var i0 = fmt.indexOf(':');
    var i1 = fmt.indexOf('.');
    var spec1 = '', spec2 = '';  // before and after dot
    if (i0 >= 0) {
        if (i1 > i0) { spec1 = fmt.slice(i0+1, i1); spec2 = fmt.slice(i1+1); }
        else { spec1 = fmt.slice(i0+1); }
    }
    // Format numbers
    if (fmt_type == '') {
    } else if (fmt_type == 'i') { // integer formatting, for %i
        s = parseInt(v).toFixed(0);
    } else if (fmt_type == 'f') {  // float formatting
        v = parseFloat(v);
        var decimals = spec2 ? Number(spec2) : 6;
        s = v.toFixed(decimals);
    } else if (fmt_type == 'e') {  // exp formatting
        v = parseFloat(v);
        var precision = (spec2 ? Number(spec2) : 6) || 1;
        s = v.toExponential(precision);
    } else if (fmt_type == 'g') {  // "general" formatting
        v = parseFloat(v);
        var precision = (spec2 ? Number(spec2) : 6) || 1;
        // Exp or decimal?
        s = v.toExponential(precision-1);
        var s1 = s.slice(0, s.indexOf('e')), s2 = s.slice(s.indexOf('e'));
        if (s2.length == 3) { s2 = 'e' + s2[1] + '0' + s2[2]; }
        var exp = Number(s2.slice(1));
        if (exp >= -4 && exp < precision) { s1=v.toPrecision(precision); s2=''; }
        // Skip trailing zeros and dot
        var j = s1.length-1;
        while (j>0 && s1[j] == '0') { j-=1; }
        s1 = s1.slice(0, j+1);
        if (s1.slice(-1) == '.') { s1 = s1.slice(0, s1.length-1); }
        s = s1 + s2;
    }
    // prefix/padding
    var prefix = '';
    if (spec1) {
        if (spec1[0] == '+' && v > 0) { prefix = '+'; spec1 = spec1.slice(1); }
        else if (spec1[0] == ' ' && v > 0) { prefix = ' '; spec1 = spec1.slice(1); }
    }
    if (spec1 && spec1[0] == '0') {
        var padding = Number(spec1.slice(1)) - (s.length + prefix.length);
        s = '0'.repeat(Math.max(0, padding)) + s;
    }
    return prefix + s;
};

var _pyfunc_int = function (x, base) { // nargs: 1 2
    if(base !== undefined) return parseInt(x, base);
    return x<0 ? Math.ceil(x): Math.floor(x);
};

var _pyfunc_op_contains = function op_contains (a, b) { // nargs: 2
    if (b == null) {
    } else if (Array.isArray(b)) {
        for (var i=0; i<b.length; i++) {if (_pyfunc_op_equals(a, b[i]))
                                           return true;}
        return false;
    } else if (b.constructor === Object) {
        for (var k in b) {if (a == k) return true;}
        return false;
    } else if (b.constructor == String) {
        return b.indexOf(a) >= 0;
    } var e = Error('Not a container: ' + b); e.name='TypeError'; throw e;
};

var _pyfunc_op_equals = function op_equals (a, b) { // nargs: 2
    var a_type = typeof a;
    // If a (or b actually) is of type string, number or boolean, we don't need
    // to do all the other type checking below.
    if (a_type === "string" || a_type === "boolean" || a_type === "number") {
        return a == b;
    }

    if (a == null || b == null) {
    } else if (Array.isArray(a) && Array.isArray(b)) {
        var i = 0, iseq = a.length == b.length;
        while (iseq && i < a.length) {iseq = op_equals(a[i], b[i]); i+=1;}
        return iseq;
    } else if (a.constructor === Object && b.constructor === Object) {
        var akeys = Object.keys(a), bkeys = Object.keys(b);
        akeys.sort(); bkeys.sort();
        var i=0, k, iseq = op_equals(akeys, bkeys);
        while (iseq && i < akeys.length)
            {k=akeys[i]; iseq = op_equals(a[k], b[k]); i+=1;}
        return iseq;
    } return a == b;
};

var _pyfunc_op_error = function (etype, msg) { // nargs: 2
    var e = new Error(etype + ': ' + msg);
    e.name = etype
    return e;
};

var _pyfunc_truthy = function (v) {
    if (v === null || typeof v !== "object") {return v;}
    else if (v.length !== undefined) {return v.length ? v : false;}
    else if (v.byteLength !== undefined) {return v.byteLength ? v : false;}
    else if (v.constructor !== Object) {return true;}
    else {return Object.getOwnPropertyNames(v).length ? v : false;}
};

var _pymeth_format = function () {
    if (this.constructor !== String) return this.format.apply(this, arguments);
    var parts = [], i = 0, i1, i2;
    var itemnr = -1;
    while (i < this.length) {
        // find opening
        i1 = this.indexOf('{', i);
        if (i1 < 0 || i1 == this.length-1) { break; }
        if (this[i1+1] == '{') {parts.push(this.slice(i, i1+1)); i = i1 + 2; continue;}
        // find closing
        i2 = this.indexOf('}', i1);
        if (i2 < 0) { break; }
        // parse
        itemnr += 1;
        var fmt = this.slice(i1+1, i2);
        var index = fmt.split(':')[0].split('!')[0];
        index = index? Number(index) : itemnr
        var s = _pyfunc_format(arguments[index], fmt);
        parts.push(this.slice(i, i1), s);
        i = i2 + 1;
    }
    parts.push(this.slice(i));
    return parts.join('');
};

var _pymeth_strip = function (chars) { // nargs: 0 1
    if (this.constructor !== String) return this.strip.apply(this, arguments);
    chars = (chars === undefined) ? ' \t\r\n' : chars;
    var i, s1 = this, s2 = '', s3 = '';
    for (i=0; i<s1.length; i++) {
        if (chars.indexOf(s1[i]) < 0) {s2 = s1.slice(i); break;}
    } for (i=s2.length-1; i>=0; i--) {
        if (chars.indexOf(s2[i]) < 0) {s3 = s2.slice(0, i+1); break;}
    } return s3;
};

var _pymeth_join = function (x) { // nargs: 1
    if (this.constructor !== String) return this.join.apply(this, arguments);
    return x.join(this);  // call join on the list instead of the string.
};

var _pyfunc_any = function (x) { // nargs: 1
    for (var i=0; i<x.length; i++) {
        if (_pyfunc_truthy(x[i])){return true;}
    } return false;
};

var _pymeth_values = function () { // nargs: 0
    if (this.constructor !== Object) return this.values.apply(this, arguments);
    var key, keys = Object.keys(this), res = [];
    for (var i=0; i<keys.length; i++) {key = keys[i]; res.push(this[key]);}
    return res;
};


var get_text;
get_text = function flx_get_text (count_of_words_to_return, letters_provided, language) {
    var ACCEPTED_LANGUAGES, MAX_WORD_LENGTH, err_2;
    MAX_WORD_LENGTH = 6;
    ACCEPTED_LANGUAGES = ["EN", "KO"];
    if ((!(_pyfunc_truthy(count_of_words_to_return) && _pyfunc_truthy(letters_provided) && _pyfunc_truthy(language)))) {
        throw _pyfunc_op_error('ValueError', _pymeth_format.call("int count expected, {} was provided; string letters expected, {letters_provided} was provided, string lang was expected, {language} was provided.", count_of_words_to_return));
    } else if ((_pyfunc_int(count_of_words_to_return) <= 0)) {
        throw _pyfunc_op_error('ValueError', _pymeth_format.call("int count must be greater than 0: {} was provided", count_of_words_to_return));
    } else if ((!_pyfunc_op_contains(language, ACCEPTED_LANGUAGES))) {
        throw _pyfunc_op_error('ValueError', _pymeth_format.call("Langauge must be one of {}: {} was provided", ACCEPTED_LANGUAGES, language));
    }
    if ((!_pyfunc_truthy(_pymeth_strip.call(letters_provided)))) {
        return " ";
    }
    if (_pyfunc_op_equals(language, "EN")) {
        return handle_EN(count_of_words_to_return, letters_provided, MAX_WORD_LENGTH);
    } else if (_pyfunc_op_equals(language, "KO")) {
        return handle_KO(count_of_words_to_return, letters_provided, MAX_WORD_LENGTH);
    } else {
        return null;
    }
    return null;
};


var handle_EN;
handle_EN = function flx_handle_EN (count_of_words_to_return, letters_provided, max_word_length) {
    var err_2, output, set_of_letters;
    set_of_letters = deduplicate_letters_EN(letters_provided);
    if ((set_of_letters.length <= 0)) {
        throw _pyfunc_op_error('ValueError', _pymeth_format.call("string letters expected at least 1 ascii letter,                         {} was provided", set_of_letters));
    }
    output = create_many_words_EN(set_of_letters, count_of_words_to_return, max_word_length);
    return _pymeth_join.call(" ", output);
};


var handle_KO;
handle_KO = function flx_handle_KO (count_of_words_to_return, letters_provided, max_word_length) {
    var dict_of_filtered_input, err_2, list_of_dict_contents_is_not_empty, output, set_of_letters, stub1_, stub1_i0, stub1_iter0, stub1_l;
    dict_of_filtered_input = filter_input_KO(letters_provided);
    stub1_ = [];stub1_iter0 = _pymeth_values.call(dict_of_filtered_input);if ((typeof stub1_iter0 === "object") && (!Array.isArray(stub1_iter0))) {stub1_iter0 = Object.keys(stub1_iter0);}for (stub1_i0=0; stub1_i0<stub1_iter0.length; stub1_i0++) {stub1_l = stub1_iter0[stub1_i0];{stub1_.push(stub1_l.length > 0);}}
    list_of_dict_contents_is_not_empty = stub1_;
    if ((!_pyfunc_any(list_of_dict_contents_is_not_empty))) {
        throw _pyfunc_op_error('ValueError', _pymeth_format.call("string letters expected at least 1 korean letter, {} was provided.", letters_provided));
    }
    set_of_letters = deduplicate_letters_KO(dict_of_filtered_input);
    output = create_many_words_KO(set_of_letters, count_of_words_to_return, max_word_length);
    return _pymeth_join.call(" ", output);
};
```



### Brython
This one actually worked, for real! By which I mean I was able to run a JavaScript version code. I did this by pasting my Python into their [web editor](https://brython.info/tests/editor.html?lang=en), and that provided JS output and ran it. It's not particularly readable to me, but neither is machine code I guess, so perhaps it doesn't need to be when compiling to something else. Compare this Python to some excepts from the JS (with some editing):

```python
def deduplicate_letters_EN(list_of_letters):
    set_of_letters = set()
    for char in list_of_letters:
        if char in ascii_letters:
            set_of_letters.add(char.lower())
    return set_of_letters
```

```javascript
  var deduplicate_letters_EN$2116 = function($defaults){
    function deduplicate_letters_EN2116(_list_of_letters){
      var $locals___main___deduplicate_letters_EN_1683 = {},
          $locals = $locals___main___deduplicate_letters_EN_1683;
      var $len = arguments.length;
      var last_arg;if($len > 0 && ((last_arg = arguments[$len - 1]) !== undefined) && last_arg.$nat !== undefined){
        $locals___main___deduplicate_letters_EN_1683 = $locals = $B.args("deduplicate_letters_EN", 1, {"list_of_letters":null}, ["list_of_letters"], arguments, $defaults, null, null);
      }else{
        if($len == 1){
          $locals___main___deduplicate_letters_EN_1683 = $locals = $B.conv_undef({"list_of_letters": _list_of_letters})
        }else if($len > 1){
          $B.wrong_nb_args("deduplicate_letters_EN", $len, 1, ["list_of_letters"])
        }else if($len + Object.keys($defaults).length < 1){
          $B.wrong_nb_args("deduplicate_letters_EN", $len, 1, ["list_of_letters"])
        }else{
          $locals___main___deduplicate_letters_EN_1683 = $locals = $B.conv_undef({"list_of_letters": _list_of_letters})
          var defparams = ["list_of_letters"]
          for(var i = $len; i < defparams.length; i++){
            $locals[defparams[i]] = $defaults[defparams[i]]
          }
        }
      }
...
    deduplicate_letters_EN2116.$is_func = true
    deduplicate_letters_EN2116.$infos = {
      __name__:"deduplicate_letters_EN",
      __qualname__:"deduplicate_letters_EN",
      __defaults__ : _b_.None,
      __kwdefaults__ : _b_.None,
      __annotations__: {},
      __dict__: $B.empty_dict(),
      __doc__: _b_.None,
      __module__ : "__main__",
      __code__:{
        co_argcount:1,
        co_filename:$locals___main__["__file__"] || "<string>",
        co_firstlineno:51,
        co_flags:67,
        co_freevars: ["ascii_letters"],
        co_kwonlyargcount:0,
        co_name: "deduplicate_letters_EN",
        co_nlocals: 3,
        co_posonlyargcount: 0,
        co_varnames: $B.fast_tuple(["list_of_letters", "set_of_letters", "char"])
      }
    };_b_.None;
    return deduplicate_letters_EN2116
```

compared to my javascript rewrite:
```javascript
function characterIsInAsciiLetterRange(letter){
    let num = ord(letter)
    return (num >= 65 && num <= 90) || (num >= 97 && num <= 122)
}

/*
 * Takes an array of letters and returns a list of
 * all letters in order
*/
function decomposeWordsEn(stringOfLetters){
    let setOfLetters = new Set()
    for (let char of stringOfLetters){
        if (characterIsInAsciiLetterRange(char)){
            setOfLetters.add(char.toLowerCase())
    }}
    return setOfLetters
}
```

That said, I'm super impressed, and think this is a great project. It also seems to have active development. I'll likely come back and dig into the explanation of [How Brython Works](https://github.com/brython-dev/brython/wiki/How%20Brython%20works).


## Final Thoughts on Transpiling
When I started out, I personally didn't think my Python code was that complicated. It's a handful of methods and lots of hashtable lookups, but no multithreading or file reading/writing or complicated class inheritance. Trying out these tools was a nice reminder that programming languages truly are complexities upon complexities. It definitely seems like a challenge to convert one set of complexities to another. 

With the way Web Assembly is going, I wouldn't be surprised to see [Python interpreters running in the browser](https://almarklein.org/python_and_webassembly.html) which seems like it would make it suuuuuuper easy to run Python, but at the expense of large file sizes.

For now, I think I'll just stick to doing the work of writing what I need.

## Next Steps
The next steps are still to make:
- logic to compare what I type to the generated words on the screen
- metrics to keep track of how well I type and which letters I should work on
- some kind of lesson structure / progression (and as part of that changing how random words are generated)

## References and Resources
- [Jest](https://jestjs.io/docs/getting-started) - [archive](https://web.archive.org/web/20220330235153/https://jestjs.io/docs/getting-started)
- [JavaScripthon](https://github.com/metapensiero/metapensiero.pj) - [archive](https://web.archive.org/web/20210605083625/https://github.com/metapensiero/metapensiero.pj)
- [RapydScript](https://github.com/atsepkov/RapydScript) - [archive](https://web.archive.org/web/20210923141219/https://github.com/atsepkov/RapydScript)
- [Transcrypt](https://github.com/QQuick/Transcrypt) - [archive](https://web.archive.org/web/20220130151249/https://github.com/QQuick/Transcrypt)
- [PScript](https://github.com/flexxui/pscript) - [archive](https://web.archive.org/web/20210827182324/https://github.com/flexxui/pscript)
- [Brython](https://github.com/brython-dev/brython) - [archive](https://web.archive.org/web/20220331181001/https://github.com/brython-dev/brython)
