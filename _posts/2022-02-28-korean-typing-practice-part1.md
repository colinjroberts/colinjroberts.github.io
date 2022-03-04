---
layout: post
title:  "Korean Typing Practice Part 1"
date:   2022-02-28 19:10:00 -0800
categories: programming
published: true
lede: "The first step of building a web tool to improve my ability to type in Korean"
---

A large part of my approach for learning Korean is accumulating vocab using [Anki](https://ankiweb.net/about). I prefer to type in my answers in Korean to help my spelling, but I am finding that I spend a lot of time hunting for keys. 

My first step is to look for something that already exists. 

**Typing tools that have Korean**
- [Korean Typing](https://www.koreantyping.com/)
- [Type Racer](https://play.typeracer.com/?universe=lang_ko)
- [Hangul Attack](https://gobillykorean.com/free-korean-typing-game-hangul-attack-new-update-2021/)


**Typing tools that don't have Korean**
- [keybr](https://www.keybr.com/)
- [Typing Club](https://www.typingclub.com/)
- [Typing Mentor](https://typingmentor.com/typing-practice/)

I've been looking for a good problem for exploring the world of web apps and APIs, so this is where I will start. All of the Korean specific typing tools I was able to find in a quick search didn't feel helpful, easy to use, or complete so this project has a real use case.


## The First Step

Looking at some of the existing tools, I have a pretty good idea of what I eventually want: a web-based tool with a simple interface that allows me to practice my Korean touch typing in an incremental fashion (i.e. start with only a few keys, and add/remove as needed) that also allows me to save my progress and to return to later.

I will need a web frontend (that I suspect will end up looking pretty similar to other typing practice tools) and a backend that will handle user auth, saving usage data nd typing stats, and possibly delivering the content for the typing lessons (though the logic is simple enough that it could all be done client side). 

I know that the front end is going to need a design and some assets, so to buy myself time to make those, I'll start on building some of the backend and business logic. I know that a key part of the app is going to be choosing which letters to practice, then generating text that use only those letters, so to get started, I'll work on building a little microservice API that can take a set of characters and return words that can be used for the typing test.


## Choice of Tools and Setup

For this first step, I want a super simple endpoint that can take some characters and return some number of words that use only those characters. This seems like an achievable chunk and like it might be helpful later on when the app needs to create arbitrary text for me to practice typing.

I'm most familiar with Python and Flask, so I'll do much of the initial work with that and can rewrite it in another language as an exercise later. Eventually, I'll want a database of some kind, but Flask and Python's built-in SQLite should be plenty for getting a prototype running. 

To get started with a Python app, my first step is always to set up a new environment. I use conda as my package manager, and I want to use Python 3.9. I try to give my environments the same name as the project folder, so I open my terminal, `cd` over to my project folder which is called `korean-typing-practice` and run the following:

```
conda create -n korean-typing-practice python=3.9
```

And I 'yes' my way through until it is ready to use, then 

```
conda activate korean-typing-practice
```


## Setting up the Flask App

**Resources Referenced Below**
- [[1] Flask Documentation](https://flask.palletsprojects.com/en/2.0.x/quickstart/#a-minimal-application) - [archive](https://web.archive.org/web/20220223144346/https://flask.palletsprojects.com/en/2.0.x/quickstart/)
- [[2] Twilio - Flask Run](https://www.twilio.com/blog/how-run-flask-application) - [archive](https://web.archive.org/web/20210507034903/https://www.twilio.com/blog/how-run-flask-application)
- [[3] Guillaume Martin - Conda Env Variables](https://guillaume-martin.github.io/saving-environment-variables-in-conda.html) - [archive](https://web.archive.org/web/20210518025536/https://guillaume-martin.github.io/saving-environment-variables-in-conda.html)

I'm going to use Flask for the first run to get things working, so I'll need to get that installed in my environment:
```
 conda install -c conda-forge flask
```

When building projects, I like to first get things working and make incremental changes, so I started with a super simple app as suggested by the [Flask Docs [1]](https://flask.palletsprojects.com/en/2.0.x/quickstart/#a-minimal-application). I made a new file called "hello.py" and filled it with the following:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello, World!'
```

Now to get it running. The Flask docs show how to save the name of the file ("hello") into a terminal variable called "FLASK_APP". I also read [a blog post from Twilio [2]](https://www.twilio.com/blog/how-run-flask-application) about Flask's run options that I will be referencing later when it comes time to put the app on a proper server. This step allows you to use Flask's built in development server for testing on your computer. Running the `export` command allows the terminal session you are using to run flask to pass that variable on to processes it spawns. I was curious as to how I might avoid having to type that in every time I start a new terminal to work on this project. 

Knowing that I might be closing and reopening terminals here and there, I wanted to find a way to avoid exporting that function every time I open a new terminal.  I followed [a tutorial by Guillaume Martin [3]](https://guillaume-martin.github.io/saving-environment-variables-in-conda.html) that documents how to add bash scripts to activate and deactivate the Flask environment variables along with the conda environment. It involves navigating to the `$CONDA_PREFIX` directory and making two directories and files. In this case `./etc/conda/activate.d/env_vars.sh`:

```bash
#!/bin/sh

export FLASK_APP=hello
export FLASK_ENV=development
```

and `./etc/conda/deactivate.d/env_vars.sh`:

```bash
#!/bin/sh

unset FLASK_APP
unset FLASK_ENV
```

Then I can deactivate and reactivate the environment and run the flask app:

```
conda deactivate
conda activate korean-typing-practice
flask run
```

Flask starts running a local development web server and gives the address `http://localhost:5000`. I type that into my web browser and I see "Hello, World!" It works! Now to start customizing. 


## Customizing the App

I make a new file `text_generator.py` that has the same structure as the sample above, but instead of " Hello, World!" it has some instructions about how to access the desired endpoint with a GET request. I'm just spitballing, but I suspect I will want to be able to provide the letters I want used in the words and the number of words to generate. I also update those conda environment variable scripts to use `text_generator` instead of `hello`. 

I know writing this for Korean might have additional problems to solve, so I'll do it for English first to protype and get the logic working. The following version of `text_generator.py` will throw an error if it doesn't get both arguments or if the number of words requested is greater than 0, otherwise it returns them.

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/')
def hello():
    return "Use the route /text with the following arguments: letters to use \
    in the words, number of words to generate. \
    e.g. localhost:5000/text?letters=abcde&count=5"

@app.route('/text', methods=['GET'])
def get_text():
    # Get arguments and return helpful errors if missing
    args = request.args.to_dict()
    count_of_words_to_return = args.get('count')
    letters_provided = args.get('letters')

    if not (count_of_words_to_return and letters_provided):
        raise ValueError(f"int count expected, {count_of_words_to_return} was \
                           provided; string letters expected, \
                           {letters_provided} was provided")
    elif int(count_of_words_to_return) <= 0:
        raise ValueError(f"int count must be greater than 0: \
                          {count_of_words_to_return} was provided")

    return args
```

With this running, the root page should have the new instructions. I can follow those instructions by changing the url to `http://localhost:5000/text?count=10&letters=deadbeef` and my browser gives me a lovely JSON version of the output (because, as [the Flask docs [1]](https://flask.palletsprojects.com/en/2.0.x/quickstart/#about-responses) explain, `jsonify()` is automatically called when returning a dict).

Now to add the fun stuff. I want to be able to take arbitrary letters provided and return some number of random "words" (they don't need to be real words, just collections of letters that I can practice typing). Working backwards, if I had a set containing one of each letter, I could randomly choose a one letter at a time to build words. Given that, I will want to deduplicate the given input, and thinking about it a little more, I will want to make sure I'm only using letters that will show up on the keyboard I'm trying to practice which is not necessarily what will be user provided. So I'll start by writing some functions that allow me to:
- check that input was provided
- deduplicate input
- filter to a particular character set (in this English case, ascii lowercase)
- then eventually use the set to generate new 'words'

```python
    # Deduplicate list of letters
    set_of_letters = set()
    for char in letters_provided:
        if char in ascii_letters:
            set_of_letters.add(char.lower())

    # Throw an error if there are no ascii letters in the set
    if len(set_of_letters) <= 0:
        raise ValueError(f"string letters expected at least 1 ascii letter, \
                        {set_of_letters} was provided")
```

I know I want to repeatedly generate words of varying length, so I need to make a function to handle the generation of a single word using a given word_length. I know I can generate random numbers, so if I have the numbers in list, I can select them by a randomly generated index.

```python
def create_one_word(list_of_letters, word_length):
    list_of_letters_for_new_word = []
    for i in range(word_length):
        random_letter = list_of_letters[randint(0, len(list_of_letters)-1)]
        list_of_letters_for_new_word.append(random_letter)
    return "".join(list_of_letters_for_new_word)

def create_many_words(set_of_letters, number_of_words, max_word_length):
    output = []
    for i in range(number_of_words):
        random_word_length = randint(1, max_word_length)
        output.append(create_one_word(list(set_of_letters), random_word_length))
    return output
```

Now I pop those functions into the app, make a variable for max word length (6 for now) and try it out. At this point, the app successfully generates words of random English letters and looks like this:

```python
from string import ascii_letters, ascii_lowercase
from flask import Flask, request
from random import randint

app = Flask(__name__)

@app.route('/')
def hello():
    return "Use the route /text with the following arguments: letters to use \
    in the words, number of words to generate. \
    e.g. localhost:5000/text?letters=abcde&count=5"

@app.route('/text', methods=['GET'])
def get_text():
    
    MAX_WORD_LENGTH = 6

    # Get arguments and return helpful errors if missing
    args = request.args.to_dict()
    count_of_words_to_return = int(args.get('count'))
    letters_provided = args.get('letters')

    if not (count_of_words_to_return and letters_provided):
        raise ValueError(f"int count expected, {count_of_words_to_return} was \
                           provided; string letters expected, \
                           {letters_provided} was provided")
    elif int(count_of_words_to_return) <= 0:
        raise ValueError(f"int count must be greater than 0: \
                          {count_of_words_to_return} was provided")

def handle_input(count_of_words_to_return, letters_provided, max_word_length):
    # Deduplicate list of letters
    set_of_letters = deduplicate_letters_EN(letters_provided)

    # Throw an error if there are no ascii letters in the set
    if len(set_of_letters) <= 0:
        raise ValueError(f"string letters expected at least 1 ascii letter, \
                        {set_of_letters} was provided")
    
    # Create and return the number of requested words
    output = create_many_words_EN(set_of_letters, count_of_words_to_return, max_word_length)
    
    return ", ".join(output)

def deduplicate_letters(list_of_letters):
    set_of_letters = set()
    for char in list_of_letters:
        if char in ascii_letters:
            set_of_letters.add(char.lower())
    return set_of_letters

def create_one_word(list_of_letters, word_length):
    list_of_letters_for_new_word = []
    for i in range(word_length):
        random_letter = list_of_letters[randint(0, len(list_of_letters)-1)]
        list_of_letters_for_new_word.append(random_letter)
    return "".join(list_of_letters_for_new_word)

def create_many_words(set_of_letters, number_of_words, max_word_length):
    output = []
    for i in range(number_of_words):
        random_word_length = randint(1, max_word_length)
        output.append(create_one_word(list(set_of_letters), random_word_length))
    return output
```
[View this as a GitHub Gist](https://gist.github.com/colinjroberts/148b251b130f6954dcb9d01dc1007401)


## Adding Korean - Research
**Resources Referenced Below**
- [[4] Wikipedia - Hangul](https://en.wikipedia.org/wiki/Hangul)
- [[5] How to Study Korean](https://www.howtostudykorean.com/unit0/)
- [[6] Talk to Me In Korean (TTMIK)](https://talktomeinkorean.com/)
- [[7] Wikipedia - Hangul in Unicode](https://en.wikipedia.org/wiki/Korean_language_and_computers#Hangul_in_Unicode) - [archive](https://web.archive.org/web/20220222181511/https://en.wikipedia.org/wiki/Korean_language_and_computers)
- [[8] Gernot Katzer's Blog](http://www.gernot-katzers-spice-pages.com/var/korean_hangul_unicode.html) - [archive](https://web.archive.org/web/20220128145936/http://gernot-katzers-spice-pages.com/var/korean_hangul_unicode.html)
- [[9] Python Docs - Unicode](https://docs.python.org/3/howto/unicode.html)
- [[10] PDF of Unicode Korean Character Charts](https://www.unicode.org/charts/PDF/UAC00.pdf)

The next step is to get it working for Korean. Though I've done my fair share of string manipulation, this is my first foray into playing around with Unicode, so I'm excited to see what challenges I get to solve. In case you aren't familiar with Korean and its writing system called [Hangul [4]](https://en.wikipedia.org/wiki/Hangul), the brief summary is that it is phonetic alphabet where each letter (mostly) represents one sound. These letters are condensed into syllable blocks and those syllable blocks are either words themselves or can be combined to make words. Though the same letter is used in different blocks, where it is in the syllable affects exactly how it is drawn and how it sounds. For example, one might use the letters 'ㄴ', 'ㄹ', 'ㄱ', and 'ㄹ' to form the syllable blocks '그', '극', '를', '늙' and many more words and non-words.

For more information on Korean and Hangul, try exploring [How to Study Korean [5]](https://www.howtostudykorean.com/unit0/) and [Talk to Me In Korean (TTMIK) [6]](https://talktomeinkorean.com/). The rest of this article will assume you understand how Hangul works and (even though it might not be exactly correct) use "words"/"syllables"/"syllable blocks" interchangeably as well as "letters"/"jamo" interchangeably.

The [Wikipedia article on Hangul in Unicode [7]](https://en.wikipedia.org/wiki/Korean_language_and_computers#Hangul_in_Unicode) and [Gernot Katzer's Blog [8]](http://www.gernot-katzers-spice-pages.com/var/korean_hangul_unicode.html) proved to be helpful resources. They explained that the Unicode tables for Korean conveniently already have every possible syllable, so one approach to converting the English word generator code to Korean could be something along the lines of deriving which characters use only the letters provided in the request, and choosing randomly from that set.

Some questions popping into my head as I start thinking about this:
- Do Python string methods work for Korean? If so, how e.g. Does `"한글".find("ㅎ")` work? Does `"한글".find("ㄹ")` return 5 (because it is the sixth letter) or 1 (because it is the second unicode character)? 
- How long are those characters? `len("ㅎ")`, `len("한글")`, `len(한글 좋아해요)`
- Along those lines, how does string slicing work? My first guess is that it operates over unicode characters such that `"한글 좋아해요"[:2] = "한글"`
- If someone is going to type in the Korean letters they want to use, will they type in each one separately pressing space after each? will they just type and let whatever syllables form? Will they type in the names of each letter)? (Almost certainly not.). This could be a moot point for the typing app, because they API requests will be automated, but it is an interesting idea to better understand.

Running these tests is left as an exercise to the reader. Below I will share some of what I've learned about Hangul and Unicode along with my approach.

The two references also describe how syllable blocks can be algorithmically generated using mapped numbers for one initial consonant, one medial vowel, then zero or one terminal consonant and plugging those into the equation `[(initial) × 588 + (medial) × 28 + (terminal)] + 44032`. This seems like a workable path for generating options! If provided a list of individual letters, I could generate all of the permutations, calculate the Unicode characters, and create a set of characters to choose from. The reverse should also be calculable because the initial and medial characters will be multiples of 28, so with a little mod(28)-ing, we can work our way backwards to decompose syllable blocks as needed. There will likely need to be a little bit of extra handling for individual characters as it seems like is more than one way to encode them, but this is a really good start!

## Changing to Korean - Decomposing and Deduplicating

Before I go any further, I want to mention that in many cases, I should look to see if someone has already written a software solution to handle all of this for me, but the fun is in the figuring out, so I am going to build something, then go look for existing solutions to compare my work to.

With that out of the way, I'm inclined to start working through the changes from the top down. The error checking leading up to `handle_input` is going to be the same, so is the first step is to work on decomposing the Korean input into a list of characters / keys that will be pressed on the keyboard. Now I don't want to completely overwrite my English work (which might be helpful later for debugging), so I'm going to renaming the functions I've written to include "EN" in the name and writing new functions for Korean that have "KO" in the name. The structure will look something like this:

```python
def handle_KO(letter_input):
    """Deduplicates Korean input by filtering for Korean Unicode codepoints 
       then deconstructing words into individual letters."""

    # Filter letter_input for only KO codepoints
    filtered_input = filter_input_KO(letter_input)

    # Decompose filtered_input into a list of characters
    decomposed_letters = decompose_words(filtered_input)

    # Convert decomposed list of letters to a set
    set_of_letters = set(decomposed_letters)

    return set_of_letters


def filter_input_KO(letter_input):
    """Filters input to only modern Korean syllables and letters
    
    Note that only modern Korean is expected so this checks only for 
    jamo (AC00–D7A3), hangul syllables (1100–11FF), and compatability jamo(3130–318F). 
    See https://www.unicode.org/faq/korean.html for more details.
    """

    # Save Unicode ranges for each type of character
    jamo_range = (int('0x1100', 16), int('0x11FF', 16))
    syllable_range = (int('0xAC00', 16), int('0xD7A3', 16))
    compatibility_jamo_range = (int('0x3130', 16), int('0x318F', 16))
    hangul_ranges = [jamo_range, syllable_range, compatibility_jamo_range]

    # Check that each input item appears in at least one of the three
    # unicode ranges
    output = []
    for item in letter_input:
        item_in_hangul_range = [range_min <= ord(item) <= range_max for range_min, range_max in hangul_ranges]
        if any(item_in_hangul_range):
            output.append(item)

    return output


def decompose_words(list_of_filtered_input):
    """Takes a list of Korean letters and words, and returns a list of all 
       letters used in those words
    """

    # Implementation to come 
    pass
```

The `decompose_words()` function is interesting because it needs to do a few things (1) Break words into the component letters and sometimes (2) when one of the component medial or terminal syllables has two letters in it (e.g. ㅝ or ㅀ ), those need to be further broken into component parts. On top of that, I'd like the resulting list to (3) be just one encoding of jamo, not a mix of composable (initial, medial, and terminal) and compatibility. The actual decomposition of a word into characters can be done with some simple calculations, and the other two points can happen quickly by establishing a letter lookup and having a function return the needed letter.

```python
def lookup_jamo(letter_block):
    """Converts all letters and multi-letter Unicode characters into a list 
    of initial or medial compatibility jamo.

    Lookup keys are Unicode Hangul Jamo (1100–11FF). Their values are lists 
    of Unicode Hangul Compatibility Jamo which have only one representation 
    per character. For example, the inital jamo 'ᄀ' (U+1100) and the terminal 
    jamo 'ᆨ' (U+11a8) will both become compatibility jamo "ㄱ" (U+3131). If 
    a character isn't a compatibility jamo and isn't in the lookup, an error 
    is thrown. 

    N.B. To be more efficient, this table could be set as a constant and 
    referenced later.
    """

    lookup = {
        'ᄀ': ["ㄱ"],
        'ᄁ': ["ㄲ"],
        'ᄂ': ["ㄴ"],
        'ᄃ': ["ㄷ"],
        'ᄄ': ["ㄸ"],
        'ᄅ': ["ㄹ"],
        'ᄆ': ["ㅁ"],
        'ᄇ': ["ㅂ"],
        'ᄈ': ["ㅃ"],
        'ᄉ': ["ㅅ"],
        'ᄊ': ["ㅆ"],
        'ᄋ': ["ㅇ"],
        'ᄌ': ["ㅈ"],
        'ᄍ': ["ㅉ"],
        'ᄎ': ["ㅊ"],
        'ᄏ': ["ㅋ"],
        'ᄐ': ["ㅌ"],
        'ᄑ': ["ㅍ"],
        'ᄒ': ["ㅎ"],
        'ᅡ': ["ㅏ"],
        'ᅢ': ["ㅐ"],
        'ᅣ': ["ㅑ"],
        'ᅤ': ["ㅒ"],
        'ᅥ': ["ㅓ"],
        'ᅦ': ["ㅔ"],
        'ᅧ': ["ㅕ"],
        'ᅨ': ["ㅖ"],
        'ᅩ': ["ㅗ"],
        'ᅪ': ["ㅗ", "ㅏ"],
        'ᅫ': ["ㅗ", "ㅐ"],
        'ᅬ': ["ㅗ", "ㅣ"],
        'ᅭ': ["ㅛ"],
        'ᅮ': ["ㅜ"],
        'ᅯ': ["ㅜ", "ㅓ"],
        'ᅰ': ["ㅜ", "ㅔ"],
        'ᅱ': ["ㅜ", "ㅣ"],
        'ᅲ': ["ㅠ"],
        'ᅳ': ["ㅡ"],
        'ᅴ': ["ㅣ"],
        'ᅵ': ["ㅣ"],
        'ᆨ': ["ㄱ"],
        'ᆩ': ["ㄲ"],
        'ᆪ': ["ㄱ", "ㅅ"],
        'ᆫ': ["ㄴ"],
        'ᆬ': ["ㄴ", "ㅈ"],
        'ᆭ': ["ㄴ", "ㅎ"],
        'ᆮ': ["ᄃ"],
        'ᆯ': ["ㄹ"],
        'ᆰ': ["ㄹ", "ㄱ"],
        'ᆱ': ["ㄹ", "ㅁ"],
        'ᆲ': ["ㄹ", "ㅂ"],
        'ᆳ': ["ㄹ", "ㅅ"],
        'ᆴ': ["ㄹ", "ㅌ"],
        'ᆵ': ["ㄹ", "ㅍ"],
        'ᆶ': ["ㄹ", "ㅎ"],
        'ᆷ': ["ㅁ"],
        'ᆸ': ["ㅂ"],
        'ᆹ': ["ㅂ", "ㅅ"],
        'ᆺ': ["ㅅ"],
        'ᆻ': ["ㅆ"],
        'ᆼ': ["ㅇ"],
        'ᆽ': ["ㅈ"],
        'ᆾ': ["ㅊ"],
        'ᆿ': ["ㅋ"],
        'ᇀ': ["ㅌ"],
        'ᇁ': ["ㅍ"],
        'ᇂ': ["ㅎ"],
    }

    output_list = lookup.get(letter_block, [])

    return output_list


def decompose_words(list_of_filtered_input):
    """Takes a list of Korean letters and words, and returns a list of
    all letters in order
    """

    output = []
    for item in list_of_filtered_input:

        # Save hex offsets for initial, medial, and terminal characters
        intial_chr_ref = 4351  # Initial hangul characters start after '0x10FF'
        mid_chr_ref = 4448  # Initial hangul characters start after '0x1161'
        terminal_chr_ref = 4519  # Initial hangul characters start after '0x11A8'

        # Calculate relative position of each jamo
        terminal = (ord(item) - 44032) % 28
        mid = 1 + ((ord(item) - 44032 - terminal) % 588 // 28)
        initial = 1 + ((ord(item) - 44032 + 1) // 588)

        # Calculate base 10 number of each Unicode Jamo
        terminal = terminal_chr_ref + terminal
        mid = mid_chr_ref + mid
        initial = intial_chr_ref + initial

        # Convert to character, then to compatibility jamo
        jamo = [chr(initial), chr(mid), chr(terminal)]
        for j in jamo:
            if j:
                output.extend(lookup_jamo(j))

    return output
```

Now this works pretty well, but decomposition only works for fully formed syllable blocks. Sending in individual jamo of any kind causes errors. So what's needed is three different code paths, one for each kind of Korean character option. `filter_input_KO()` is the first step of processing input, and it is already comparing each input character to all three sets of Korean characters, so I might as well rewrite that to sort the characters into different buckets that I can use later. To do that, I will change it from returning one list to returning a dict with with 3 lists, one each for jamo, syllables, and compatibility jamo. The result is that a call like `deduplicate_letters_KO("ᄑㅎㅏᇀ아")` will have an output of `{'jamo_range': ['아'], 'syllable_range': ['ᄑ', 'ᇀ'], 'compatibility_jamo_range': ['ㅎ', 'ㅏ']}`. The revised function looks like this:

```python
def filter_input_KO(letter_input):
    """Converts all letters and multi-letter Unicode characters into a list 
    of initial or medial compatibility jamo.

    Lookup keys are Unicode Hangul Jamo (1100–11FF). Their values are lists 
    of Unicode Hangul Compatibility Jamo which have only one representation 
    per character. For example, the inital jamo 'ᄀ' (U+1100) and the terminal 
    jamo 'ᆨ' (U+11a8) will both become compatibility jamo "ㄱ" (U+3131). If 
    a character isn't a compatibility jamo and isn't in the lookup, an error 
    is thrown. 

    N.B. To be more efficient, this table could be set as a constant and 
    referenced later.
    """

    hangul_ranges = {
        "jamo": (int('0x1100', 16), int('0x11FF', 16)),
        "syllable": (int('0xAC00', 16), int('0xD7A3', 16)),
        "compatibility_jamo": (int('0x3130', 16), int('0x318F', 16)),
    }

    output = {"jamo": [], "syllable": [], "compatibility_jamo": []}
    for item in letter_input:
        for (key, (range_min, range_max)) in hangul_ranges.items():
            if range_min <= ord(item) <= range_max:
                output[key].append(item)

    return output
```

Now, we can take the appropriate conversion for each character! Here is the revised main deduplication function that filters for the three kinds of modern Korean Unicode characters, then converts or adds only compatibility jamo to the output, and terminally returns a set of those characters.

```python
def deduplicate_letters_KO(dict_of_filtered_input):
    # Decompose letter_input into characters
    decomposed_letters = []
    decomposed_letters.extend(dict_of_filtered_input["compatibility_jamo"])
    for j in dict_of_filtered_input["jamo"]:
        decomposed_letters.extend(lookup_jamo(j))
    for s in dict_of_filtered_input["syllable"]:
        decomposed_letters.extend(decompose_words(s))

    # Convert decomposed list of letters to a set
    set_of_letters = set(decomposed_letters)

    return set_of_letters
```

Put that all together and we should have an app that can take arbitrary Korean letters and return a deduplicated list of compatibility jamo. To tidy up the code, we will add a parameter to the GET request for language that will accept either EN or KO.

Note that later in the code, we will want to check to make sure we have enough letters to make words. If there are no vowels, we can't make full words so we should handle that case. In reality though, when texting and typing, there are [some situations](https://www.reddit.com/r/Korean/comments/ajektl/korean_chattinginternet_slangs/) where you might not be writing whole words. 


```python
from string import ascii_letters, ascii_lowercase
from flask import Flask, request
from random import randint

app = Flask(__name__)

@app.route('/')
def hello():
    return "Use the route /text with the following arguments: letters to use \
    in the words, number of words to generate. \
    e.g. localhost:5000/text?letters=abcde&count=5"
    
@app.route('/text', methods=['GET'])
def get_text():
    MAX_WORD_LENGTH = 6
    ACCEPTED_LANGUAGES = ('EN', 'KO')

    # Get arguments and return helpful errors if missing
    args = request.args.to_dict()
    count_of_words_to_return = int(args.get('count'))
    letters_provided = args.get('letters')
    language = args.get('lang')

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
    
    return ", ".join(output)


def deduplicate_letters_EN(list_of_letters):
    set_of_letters = set()
    for char in list_of_letters:
        if char in ascii_letters:
            set_of_letters.add(char.lower())
    return set_of_letters


def create_one_word_EN(list_of_letters, word_length):
    list_of_letters_for_new_word = []
    for i in range(word_length):
        random_letter = list_of_letters[randint(0, len(list_of_letters)-1)]
        list_of_letters_for_new_word.append(random_letter)
    return "".join(list_of_letters_for_new_word)


def create_many_words_EN(set_of_letters, number_of_words, max_word_length):
    output = []
    for i in range(number_of_words):
        random_word_length = randint(1, max_word_length)
        output.append(create_one_word_EN(list(set_of_letters), random_word_length))
    return output


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

    return ", ".join(set_of_letters)
    
    
def deduplicate_letters_KO(dict_of_filtered_input):
    # Decompose letter_input into characters
    decomposed_letters = []
    decomposed_letters.extend(dict_of_filtered_input["compatibility_jamo"])
    for j in dict_of_filtered_input["jamo"]:
        decomposed_letters.extend(lookup_jamo_KO(j))
    for s in dict_of_filtered_input["syllable"]:
        decomposed_letters.extend(decompose_words_KO(s))

    # Convert decomposed list of letters to a set
    set_of_letters = set(decomposed_letters)

    return set_of_letters


def filter_input_KO(letter_input):
    """Converts all letters and multi-letter Unicode characters into a list 
    of initial or medial compatibility jamo.

    Lookup keys are Unicode Hangul Jamo (1100–11FF). Their values are lists 
    of Unicode Hangul Compatibility Jamo which have only one representation 
    per character. For example, the inital jamo 'ᄀ' (U+1100) and the terminal 
    jamo 'ᆨ' (U+11a8) will both become compatibility jamo "ㄱ" (U+3131). If 
    a character isn't a compatibility jamo and isn't in the lookup, an error 
    is thrown. 

    N.B. To be more efficient, this table could be set as a constant and 
    referenced later.
    """

    hangul_ranges = {
        "jamo": (int('0x1100', 16), int('0x11FF', 16)),
        "syllable": (int('0xAC00', 16), int('0xD7A3', 16)),
        "compatibility_jamo": (int('0x3130', 16), int('0x318F', 16)),
    }

    output = {"jamo": [], "syllable": [], "compatibility_jamo": []}
    for item in letter_input:
        for (key, (range_min, range_max)) in hangul_ranges.items():
            if range_min <= ord(item) <= range_max:
                output[key].append(item)

    return output


def decompose_words_KO(list_of_filtered_input):
    """Takes a list of Korean letters and words, and returns a list of
    all letters in order
    """

    output = []
    for item in list_of_filtered_input:

        # Save hex offsets for initial, medial, and terminal characters
        intial_chr_ref = 4351  # Initial hangul characters start after '0x10FF'
        mid_chr_ref = 4448  # Initial hangul characters start after '0x1161'
        terminal_chr_ref = 4519  # Initial hangul characters start after '0x11A8'

        # Calculate relative position of each jamo
        terminal = (ord(item) - 44032) % 28
        mid = 1 + ((ord(item) - 44032 - terminal) % 588 // 28)
        initial = 1 + ((ord(item) - 44032 + 1) // 588)

        # Calculate base 10 number of each Unicode Jamo
        terminal = terminal_chr_ref + terminal
        mid = mid_chr_ref + mid
        initial = intial_chr_ref + initial

        # Convert to character, then to compatibility jamo
        jamo = [chr(initial), chr(mid), chr(terminal)]
        for j in jamo:
            if j:
                output.extend(lookup_jamo_KO(j))

    return output


def lookup_jamo_KO(letter_block):
    """Converts all letters and multi-letter Unicode characters into a list 
    of initial or medial compatibility jamo.

    Lookup keys are Unicode Hangul Jamo (1100–11FF). Their values are lists 
    of Unicode Hangul Compatibility Jamo which have only one representation 
    per character. For example, the inital jamo 'ᄀ' (U+1100) and the terminal 
    jamo 'ᆨ' (U+11a8) will both become compatibility jamo "ㄱ" (U+3131). If 
    a character isn't a compatibility jamo and isn't in the lookup, an error 
    is thrown. 

    N.B. To be more efficient, this table could be set as a constant and 
    referenced later.
    """

    lookup = {
        'ᄀ': ["ㄱ"],
        'ᄁ': ["ㄲ"],
        'ᄂ': ["ㄴ"],
        'ᄃ': ["ㄷ"],
        'ᄄ': ["ㄸ"],
        'ᄅ': ["ㄹ"],
        'ᄆ': ["ㅁ"],
        'ᄇ': ["ㅂ"],
        'ᄈ': ["ㅃ"],
        'ᄉ': ["ㅅ"],
        'ᄊ': ["ㅆ"],
        'ᄋ': ["ㅇ"],
        'ᄌ': ["ㅈ"],
        'ᄍ': ["ㅉ"],
        'ᄎ': ["ㅊ"],
        'ᄏ': ["ㅋ"],
        'ᄐ': ["ㅌ"],
        'ᄑ': ["ㅍ"],
        'ᄒ': ["ㅎ"],
        'ᅡ': ["ㅏ"],
        'ᅢ': ["ㅐ"],
        'ᅣ': ["ㅑ"],
        'ᅤ': ["ㅒ"],
        'ᅥ': ["ㅓ"],
        'ᅦ': ["ㅔ"],
        'ᅧ': ["ㅕ"],
        'ᅨ': ["ㅖ"],
        'ᅩ': ["ㅗ"],
        'ᅪ': ["ㅗ", "ㅏ"],
        'ᅫ': ["ㅗ", "ㅐ"],
        'ᅬ': ["ㅗ", "ㅣ"],
        'ᅭ': ["ㅛ"],
        'ᅮ': ["ㅜ"],
        'ᅯ': ["ㅜ", "ㅓ"],
        'ᅰ': ["ㅜ", "ㅔ"],
        'ᅱ': ["ㅜ", "ㅣ"],
        'ᅲ': ["ㅠ"],
        'ᅳ': ["ㅡ"],
        'ᅴ': ["ㅣ"],
        'ᅵ': ["ㅣ"],
        'ᆨ': ["ㄱ"],
        'ᆩ': ["ㄲ"],
        'ᆪ': ["ㄱ", "ㅅ"],
        'ᆫ': ["ㄴ"],
        'ᆬ': ["ㄴ", "ㅈ"],
        'ᆭ': ["ㄴ", "ㅎ"],
        'ᆮ': ["ᄃ"],
        'ᆯ': ["ㄹ"],
        'ᆰ': ["ㄹ", "ㄱ"],
        'ᆱ': ["ㄹ", "ㅁ"],
        'ᆲ': ["ㄹ", "ㅂ"],
        'ᆳ': ["ㄹ", "ㅅ"],
        'ᆴ': ["ㄹ", "ㅌ"],
        'ᆵ': ["ㄹ", "ㅍ"],
        'ᆶ': ["ㄹ", "ㅎ"],
        'ᆷ': ["ㅁ"],
        'ᆸ': ["ㅂ"],
        'ᆹ': ["ㅂ", "ㅅ"],
        'ᆺ': ["ㅅ"],
        'ᆻ': ["ㅆ"],
        'ᆼ': ["ㅇ"],
        'ᆽ': ["ㅈ"],
        'ᆾ': ["ㅊ"],
        'ᆿ': ["ㅋ"],
        'ᇀ': ["ㅌ"],
        'ᇁ': ["ㅍ"],
        'ᇂ': ["ㅎ"],
    }

    output_list = lookup.get(letter_block, [])

    return output_list
```
[View this as a GitHub Gist](https://gist.github.com/colinjroberts/6e48bfb4ae664a3eb1527d03c5bcad87)


## Changing to Korean - Composing New Random Syllables

Now that I have a deduplicated list of letters to use, I can reimplement the create one word and create many words functions for Korean. In the English version, I took the approach of randomly selecting letters by using `randint`. I could take a similar approach to what I did in English by randomly selecting from a list of possible characters. Right away, I'm cautious of this approach, because as the set of input characters increases, the size of list of possible syllable blocks would increase until it reaches the complete list of all 11,172 possible syllables. 

A better option is to use the same approach that Unicode does to compose syllables as needed. I could make lists of initial, medial, and terminal letters (having converted them from the compatibility jamo to an index number used specifically for this purpose), then randomly choose one from each list to make syllables and avoid ever having to load the complete set. Here's the rough outline:

```python
def create_jamo_lists(set_of_compatibility_jamo):
    # Lookup Unicode refs for initial
    initial_jamo_list = []

    # Lookup Unicode refs for medial
    medial_jamo_list = []

    # Lookup Unicode refs for terminal
    terminal_jamo_list = []

    return initial_jamo_list, medial_jamo_list, terminal_jamo_list


def create_word_KO(initial_set, medial_set, terminal_set, number_of_syllables):
    list_of_syllable_blocks = []
    for i in range(number_of_syllables):
        # Randomly choose from initial set

        # Randomly choose from medial set

        # Randomly choose from terminal set

        syllable_block = None # Calculate Unicode for syllable block

        list_of_syllable_blocks.append(syllable_block)

    return "".join(list_of_syllable_blocks)
```

I'll start with `create_jamo_lists()` which is going to be a kind of opposite to the `lookup_jamo()` from earlier, but instead of converting to an actual character, I'm going to convert directly to an index number. It takes an iterable of compatibility jamo and returns three lists of character indices that will later be used to assemble syllables. It is also going to need to handle all of the combined jamo sets (e.g. including 'ㅟ' when 'ㅜ' and 'ㅣ' are both present). My first inclination is to split each one into its own function, but given that there shouldn't be much need for this elsewhere and because the logic shouldn't need to change, I'm going to keep it as one big function.

```python
def create_jamo_lists(set_of_compatibility_jamo):

    # Set up initial character dict (compatibility jamo -> initial jamo)
    initial_char_dict = {
        'ㄱ': 0,  # 'ᄀ',
        'ㄲ': 1,  # 'ᄁ',
        'ㄴ': 2,  # 'ᄂ',
        'ㄷ': 3,  # 'ᄃ',
        'ㄸ': 4,  # 'ᄄ',
        'ㄹ': 5,  # 'ᄅ',
        'ㅁ': 6,  # 'ᄆ',
        'ㅂ': 7,  # 'ᄇ',
        'ㅃ': 8,  # 'ᄈ',
        'ㅅ': 9,  # 'ᄉ',
        'ㅆ': 10,  # 'ᄊ',
        'ㅇ': 11,  # 'ᄋ',
        'ㅈ': 12,  # 'ᄌ',
        'ㅉ': 13,  # 'ᄍ',
        'ㅊ': 14,  # 'ᄎ',
        'ㅋ': 15,  # 'ᄏ',
        'ㅌ': 16,  # 'ᄐ',
        'ㅍ': 17,  # 'ᄑ',
        'ㅎ': 18,  # 'ᄒ',
    }

    # Set up medial character dict (compatibility jamo -> medial jamo)
    medial_char_dict = {
        'ㅏ': 0,  # 'ᅡ',
        'ㅐ': 1,  # 'ᅢ',
        'ㅑ': 2,  # 'ᅣ',
        'ㅒ': 3,  # 'ᅤ',
        'ㅓ': 4,  # 'ᅥ',
        'ㅔ': 5,  # 'ᅦ',
        'ㅕ': 6,  # 'ᅧ',
        'ㅖ': 7,  # 'ᅨ',
        'ㅗ': 8,  # 'ᅩ',
        'ㅛ': 12,  # 'ᅭ',
        'ㅜ': 13,  # 'ᅮ',
        'ㅠ': 17,  # 'ᅲ',
        'ㅡ': 18,  # 'ᅳ',
        'ㅣ': 20,  # 'ᅵ',
    }

    # Set up terminal character dict (compatibility jamo -> terminal jamo)
    terminal_char_dict = {
        'ㄱ': 1,  #'ᆨ',
        'ㄲ': 2,  #'ᆩ',
        'ㄴ': 4,  #'ᆫ',
        'ㄷ': 7,  #'ᆮ',
        'ㄹ': 8,  #'ᆯ',
        'ㅁ': 16,  #'ᆷ',
        'ㅂ': 17,  #'ᆸ',
        'ㅅ': 19,  #'ᆺ',
        'ㅆ': 20,  #'ᆻ',
        'ㅇ': 21,  #'ᆼ',
        'ㅈ': 22,  #'ᆽ',
        'ㅊ': 23,  #'ᆾ',
        'ㅋ': 24,  #'ᆿ',
        'ㅌ': 25,  #'ᇀ',
        'ㅍ': 26,  #'ᇁ',
        'ㅎ': 27,  #'ᇂ',
    }

    initial_jamo_list = []
    medial_jamo_list = []
    terminal_jamo_list = [0]  # Syllable blocks can be only two letters long, so a 0 option is needed for none

    # Look up Unicode refs for single character initial, medial, and terminal
    for item in set_of_compatibility_jamo:
        if item in initial_char_dict:
            initial_jamo_list.append(initial_char_dict[item])
        if item in medial_char_dict:
            medial_jamo_list.append(medial_char_dict[item])
        if item in terminal_char_dict:
            terminal_jamo_list.append(terminal_char_dict[item])

    # Add composite medial chars if needed
    if 'ㅗ' in set_of_compatibility_jamo:
        if 'ㅏ' in set_of_compatibility_jamo:
            medial_jamo_list.append(9)  # 'ᅪ'
        if 'ㅐ' in set_of_compatibility_jamo:
            medial_jamo_list.append(10)  # 'ᅫ'
        if 'ㅣ' in set_of_compatibility_jamo:
            medial_jamo_list.append(11)  # 'ᅬ'

    if 'ㅜ' in set_of_compatibility_jamo:
        if 'ㅓ' in set_of_compatibility_jamo:
            medial_jamo_list.append(14)  # 'ᅯ'
        if 'ㅔ' in set_of_compatibility_jamo:
            medial_jamo_list.append(15)  # 'ᅰ'
        if 'ㅣ' in set_of_compatibility_jamo:
            medial_jamo_list.append(16)  # 'ᅱ'

    if 'ㅡ' in set_of_compatibility_jamo:
        if 'ㅣ' in set_of_compatibility_jamo:
            medial_jamo_list.append(19)  # 'ᅴ'

    # Add composite terminal chars if needed
    if 'ㄱ' in set_of_compatibility_jamo:
        if 'ㅅ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(3)  # 'ᆪ'

    if 'ㄴ' in set_of_compatibility_jamo:
        if 'ㅈ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(5)  # 'ᆬ'
        if 'ㅎ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(6)  # 'ᆭ'

    if 'ㄹ' in set_of_compatibility_jamo:
        if 'ㄱ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(9)  # 'ᆰ'
        if 'ㅁ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(10)  # 'ᆱ'
        if 'ㅂ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(11)  # 'ᆲ'
        if 'ㅅ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(12)  # 'ᆳ'
        if 'ㅌ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(13)  # 'ᆴ'
        if 'ㅍ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(14)  # 'ᆵ'
        if 'ㅎ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(15)  # 'ᆶ'

    if 'ㅂ' in set_of_compatibility_jamo:
        if 'ㅅ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(18)  # 'ᆹ'

    return initial_jamo_list, medial_jamo_list, terminal_jamo_list
```

Now, I can use the sets of characters to start randomly making words to type. The following two functions repeat similar logic to the English versions by randomly selecting from each list, but in this case it then gets transformed into a Korean syllable block.

```python
def create_one_word_KO(initial_list, medial_list, terminal_list, number_of_syllables):

    list_of_syllable_blocks = []
    for i in range(number_of_syllables):

        # Randomly choose from initial set
        initial_letter = initial_list[randint(0, len(initial_list)-1)]

        # Randomly choose from medial set
        medial_letter = medial_list[randint(0, len(medial_list)-1)]

        # Randomly choose from terminal set
        terminal_letter = terminal_list[randint(0, len(terminal_list)-1)]

        # Calculate Unicode for syllable block
        syllable_block = (initial_letter * 588) + (medial_letter * 28) + terminal_letter + 44032  

        list_of_syllable_blocks.append(chr(syllable_block))

    return "".join(list_of_syllable_blocks)


def create_many_words_KO(set_of_compatibility_jamo, number_of_words, max_word_length):
    # Lookup lists of Unicode refs for initial, medial, and terminal
    initial_jamo_list, medial_jamo_list, terminal_jamo_list = create_jamo_lists(set_of_compatibility_jamo)

    # Throw error if no initial or medial jamo exist
    if not (initial_jamo_list and medial_jamo_list):
        raise ValueError(f"There must be at least one initial letter and at least one vowel.")

    # Create a list of words
    output = []
    for i in range(number_of_words):
        output.append(create_one_word_KO(initial_jamo_list, medial_jamo_list, terminal_jamo_list, randint(1,max_word_length)))

    return output
```

The last step is to plug all of those functions into the Flask app, tidy things up, and make sure everything is connected. Here is the final working code that takes input via HTTP parameters such that a request like `http://localhost:5000/text?count=5&letters=hahahoho하하호호&lang=EN` produces output like `hhhh, hhoh, oho, hhohhh, ahoao` and `http://localhost:5000/text?count=5&letters=hahahoho하하호호&lang=KO` produces output like `홯홯화홓홓, 호, 호호하, 홓홯홯, 호홓홓화화`.

```python
from string import ascii_letters, ascii_lowercase
from flask import Flask, request
from random import randint

app = Flask(__name__)

@app.route('/')
def hello():
    return "Use the route /text with the following arguments: letters to use \
    in the words, number of words to generate. \
    e.g. localhost:5000/text?letters=abcde&count=5"
    
@app.route('/text', methods=['GET'])
def get_text():
    
    MAX_WORD_LENGTH = 6
    ACCEPTED_LANGUAGES = ('EN', 'KO')

    # Get arguments and return helpful errors if missing
    args = request.args.to_dict()
    count_of_words_to_return = int(args.get('count'))
    letters_provided = args.get('letters')
    language = args.get('lang')

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
    
    return ", ".join(output)


def deduplicate_letters_EN(list_of_letters):
    set_of_letters = set()
    for char in list_of_letters:
        if char in ascii_letters:
            set_of_letters.add(char.lower())
    return set_of_letters


def create_one_word_EN(list_of_letters, word_length):
    list_of_letters_for_new_word = []
    for i in range(word_length):
        random_letter = list_of_letters[randint(0, len(list_of_letters)-1)]
        list_of_letters_for_new_word.append(random_letter)
    return "".join(list_of_letters_for_new_word)


def create_many_words_EN(set_of_letters, number_of_words, max_word_length):
    output = []
    for i in range(number_of_words):
        random_word_length = randint(1, max_word_length)
        output.append(create_one_word_EN(list(set_of_letters), random_word_length))
    return output


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
    
    return ", ".join(output)
    
    
def deduplicate_letters_KO(dict_of_filtered_input):
    # Decompose letter_input into characters
    decomposed_letters = []
    decomposed_letters.extend(dict_of_filtered_input["compatibility_jamo"])
    for j in dict_of_filtered_input["jamo"]:
        decomposed_letters.extend(lookup_jamo_KO(j))
    for s in dict_of_filtered_input["syllable"]:
        decomposed_letters.extend(decompose_words_KO(s))

    # Convert decomposed list of letters to a set
    set_of_letters = set(decomposed_letters)

    return set_of_letters


def filter_input_KO(letter_input):
    """Converts all letters and multi-letter Unicode characters into a list 
    of initial or medial compatibility jamo.

    Lookup keys are Unicode Hangul Jamo (1100–11FF). Their values are lists 
    of Unicode Hangul Compatibility Jamo which have only one representation 
    per character. For example, the inital jamo 'ᄀ' (U+1100) and the terminal 
    jamo 'ᆨ' (U+11a8) will both become compatibility jamo "ㄱ" (U+3131). If 
    a character isn't a compatibility jamo and isn't in the lookup, an error 
    is thrown. 

    N.B. To be more efficient, this table could be set as a constant and 
    referenced later.
    """

    hangul_ranges = {
        "jamo": (int('0x1100', 16), int('0x11FF', 16)),
        "syllable": (int('0xAC00', 16), int('0xD7A3', 16)),
        "compatibility_jamo": (int('0x3130', 16), int('0x318F', 16)),
    }

    output = {"jamo": [], "syllable": [], "compatibility_jamo": []}
    for item in letter_input:
        for (key, (range_min, range_max)) in hangul_ranges.items():
            if range_min <= ord(item) <= range_max:
                output[key].append(item)

    return output


def decompose_words_KO(list_of_filtered_input):
    """Takes a list of Korean letters and words, and returns a list of
    all letters in order
    """

    output = []
    for item in list_of_filtered_input:

        # Save hex offsets for initial, medial, and terminal characters
        intial_chr_ref = 4351  # Initial hangul characters start after '0x10FF'
        mid_chr_ref = 4448  # Initial hangul characters start after '0x1161'
        terminal_chr_ref = 4519  # Initial hangul characters start after '0x11A8'

        # Calculate relative position of each jamo
        terminal = (ord(item) - 44032) % 28
        mid = 1 + ((ord(item) - 44032 - terminal) % 588 // 28)
        initial = 1 + ((ord(item) - 44032 + 1) // 588)

        # Calculate base 10 number of each Unicode Jamo
        terminal = terminal_chr_ref + terminal
        mid = mid_chr_ref + mid
        initial = intial_chr_ref + initial

        # Convert to character, then to compatibility jamo
        jamo = [chr(initial), chr(mid), chr(terminal)]
        for j in jamo:
            if j:
                output.extend(lookup_jamo_KO(j))

    return output


def lookup_jamo_KO(letter_block):
    """Converts all letters and multi-letter Unicode characters into a list 
    of initial or medial compatibility jamo.

    Lookup keys are Unicode Hangul Jamo (1100–11FF). Their values are lists 
    of Unicode Hangul Compatibility Jamo which have only one representation 
    per character. For example, the inital jamo 'ᄀ' (U+1100) and the terminal 
    jamo 'ᆨ' (U+11a8) will both become compatibility jamo "ㄱ" (U+3131). If 
    a character isn't a compatibility jamo and isn't in the lookup, an error 
    is thrown. 

    N.B. To be more efficient, this table could be set as a constant and 
    referenced later.
    """

    lookup = {
        'ᄀ': ["ㄱ"],
        'ᄁ': ["ㄲ"],
        'ᄂ': ["ㄴ"],
        'ᄃ': ["ㄷ"],
        'ᄄ': ["ㄸ"],
        'ᄅ': ["ㄹ"],
        'ᄆ': ["ㅁ"],
        'ᄇ': ["ㅂ"],
        'ᄈ': ["ㅃ"],
        'ᄉ': ["ㅅ"],
        'ᄊ': ["ㅆ"],
        'ᄋ': ["ㅇ"],
        'ᄌ': ["ㅈ"],
        'ᄍ': ["ㅉ"],
        'ᄎ': ["ㅊ"],
        'ᄏ': ["ㅋ"],
        'ᄐ': ["ㅌ"],
        'ᄑ': ["ㅍ"],
        'ᄒ': ["ㅎ"],
        'ᅡ': ["ㅏ"],
        'ᅢ': ["ㅐ"],
        'ᅣ': ["ㅑ"],
        'ᅤ': ["ㅒ"],
        'ᅥ': ["ㅓ"],
        'ᅦ': ["ㅔ"],
        'ᅧ': ["ㅕ"],
        'ᅨ': ["ㅖ"],
        'ᅩ': ["ㅗ"],
        'ᅪ': ["ㅗ", "ㅏ"],
        'ᅫ': ["ㅗ", "ㅐ"],
        'ᅬ': ["ㅗ", "ㅣ"],
        'ᅭ': ["ㅛ"],
        'ᅮ': ["ㅜ"],
        'ᅯ': ["ㅜ", "ㅓ"],
        'ᅰ': ["ㅜ", "ㅔ"],
        'ᅱ': ["ㅜ", "ㅣ"],
        'ᅲ': ["ㅠ"],
        'ᅳ': ["ㅡ"],
        'ᅴ': ["ㅣ"],
        'ᅵ': ["ㅣ"],
        'ᆨ': ["ㄱ"],
        'ᆩ': ["ㄲ"],
        'ᆪ': ["ㄱ", "ㅅ"],
        'ᆫ': ["ㄴ"],
        'ᆬ': ["ㄴ", "ㅈ"],
        'ᆭ': ["ㄴ", "ㅎ"],
        'ᆮ': ["ᄃ"],
        'ᆯ': ["ㄹ"],
        'ᆰ': ["ㄹ", "ㄱ"],
        'ᆱ': ["ㄹ", "ㅁ"],
        'ᆲ': ["ㄹ", "ㅂ"],
        'ᆳ': ["ㄹ", "ㅅ"],
        'ᆴ': ["ㄹ", "ㅌ"],
        'ᆵ': ["ㄹ", "ㅍ"],
        'ᆶ': ["ㄹ", "ㅎ"],
        'ᆷ': ["ㅁ"],
        'ᆸ': ["ㅂ"],
        'ᆹ': ["ㅂ", "ㅅ"],
        'ᆺ': ["ㅅ"],
        'ᆻ': ["ㅆ"],
        'ᆼ': ["ㅇ"],
        'ᆽ': ["ㅈ"],
        'ᆾ': ["ㅊ"],
        'ᆿ': ["ㅋ"],
        'ᇀ': ["ㅌ"],
        'ᇁ': ["ㅍ"],
        'ᇂ': ["ㅎ"],
    }

    output_list = lookup.get(letter_block, [])

    return output_list


def create_jamo_lists(set_of_compatibility_jamo):
    # Set up initial character dict (compatibility jamo -> initial jamo)
    initial_char_dict = {
        'ㄱ': 0,  # 'ᄀ',
        'ㄲ': 1,  # 'ᄁ',
        'ㄴ': 2,  # 'ᄂ',
        'ㄷ': 3,  # 'ᄃ',
        'ㄸ': 4,  # 'ᄄ',
        'ㄹ': 5,  # 'ᄅ',
        'ㅁ': 6,  # 'ᄆ',
        'ㅂ': 7,  # 'ᄇ',
        'ㅃ': 8,  # 'ᄈ',
        'ㅅ': 9,  # 'ᄉ',
        'ㅆ': 10,  # 'ᄊ',
        'ㅇ': 11,  # 'ᄋ',
        'ㅈ': 12,  # 'ᄌ',
        'ㅉ': 13,  # 'ᄍ',
        'ㅊ': 14,  # 'ᄎ',
        'ㅋ': 15,  # 'ᄏ',
        'ㅌ': 16,  # 'ᄐ',
        'ㅍ': 17,  # 'ᄑ',
        'ㅎ': 18,  # 'ᄒ',
    }

    # Set up medial character dict (compatibility jamo -> medial jamo)
    medial_char_dict = {
        'ㅏ': 0,  # 'ᅡ',
        'ㅐ': 1,  # 'ᅢ',
        'ㅑ': 2,  # 'ᅣ',
        'ㅒ': 3,  # 'ᅤ',
        'ㅓ': 4,  # 'ᅥ',
        'ㅔ': 5,  # 'ᅦ',
        'ㅕ': 6,  # 'ᅧ',
        'ㅖ': 7,  # 'ᅨ',
        'ㅗ': 8,  # 'ᅩ',
        'ㅛ': 12,  # 'ᅭ',
        'ㅜ': 13,  # 'ᅮ',
        'ㅠ': 17,  # 'ᅲ',
        'ㅡ': 18,  # 'ᅳ',
        'ㅣ': 20,  # 'ᅵ',
    }

    # Set up terminal character dict (compatibility jamo -> terminal jamo)
    terminal_char_dict = {
        'ㄱ': 1,  #'ᆨ',
        'ㄲ': 2,  #'ᆩ',
        'ㄴ': 4,  #'ᆫ',
        'ㄷ': 7,  #'ᆮ',
        'ㄹ': 8,  #'ᆯ',
        'ㅁ': 16,  #'ᆷ',
        'ㅂ': 17,  #'ᆸ',
        'ㅅ': 19,  #'ᆺ',
        'ㅆ': 20,  #'ᆻ',
        'ㅇ': 21,  #'ᆼ',
        'ㅈ': 22,  #'ᆽ',
        'ㅊ': 23,  #'ᆾ',
        'ㅋ': 24,  #'ᆿ',
        'ㅌ': 25,  #'ᇀ',
        'ㅍ': 26,  #'ᇁ',
        'ㅎ': 27,  #'ᇂ',
    }

    initial_jamo_list = []
    medial_jamo_list = []
    terminal_jamo_list = [0]  # Syllable blocks can be only two letters long, so a 0 option is needed for none

    # Look up Unicode refs for single character initial, medial, and terminal
    for item in set_of_compatibility_jamo:
        if item in initial_char_dict:
            initial_jamo_list.append(initial_char_dict[item])
        if item in medial_char_dict:
            medial_jamo_list.append(medial_char_dict[item])
        if item in terminal_char_dict:
            terminal_jamo_list.append(terminal_char_dict[item])

    # Add composite medial chars if needed
    if 'ㅗ' in set_of_compatibility_jamo:
        if 'ㅏ' in set_of_compatibility_jamo:
            medial_jamo_list.append(9)  # 'ᅪ'
        if 'ㅐ' in set_of_compatibility_jamo:
            medial_jamo_list.append(10)  # 'ᅫ'
        if 'ㅣ' in set_of_compatibility_jamo:
            medial_jamo_list.append(11)  # 'ᅬ'

    if 'ㅜ' in set_of_compatibility_jamo:
        if 'ㅓ' in set_of_compatibility_jamo:
            medial_jamo_list.append(14)  # 'ᅯ'
        if 'ㅔ' in set_of_compatibility_jamo:
            medial_jamo_list.append(15)  # 'ᅰ'
        if 'ㅣ' in set_of_compatibility_jamo:
            medial_jamo_list.append(16)  # 'ᅱ'

    if 'ㅡ' in set_of_compatibility_jamo:
        if 'ㅣ' in set_of_compatibility_jamo:
            medial_jamo_list.append(19)  # 'ᅴ'

    # Add composite terminal chars if needed
    if 'ㄱ' in set_of_compatibility_jamo:
        if 'ㅅ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(3)  # 'ᆪ'

    if 'ㄴ' in set_of_compatibility_jamo:
        if 'ㅈ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(5)  # 'ᆬ'
        if 'ㅎ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(6)  # 'ᆭ'

    if 'ㄹ' in set_of_compatibility_jamo:
        if 'ㄱ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(9)  # 'ᆰ'
        if 'ㅁ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(10)  # 'ᆱ'
        if 'ㅂ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(11)  # 'ᆲ'
        if 'ㅅ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(12)  # 'ᆳ'
        if 'ㅌ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(13)  # 'ᆴ'
        if 'ㅍ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(14)  # 'ᆵ'
        if 'ㅎ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(15)  # 'ᆶ'

    if 'ㅂ' in set_of_compatibility_jamo:
        if 'ㅅ' in set_of_compatibility_jamo:
            terminal_jamo_list.append(18)  # 'ᆹ'

    return initial_jamo_list, medial_jamo_list, terminal_jamo_list


def create_one_word_KO(initial_list, medial_list, terminal_list, number_of_syllables):

    list_of_syllable_blocks = []
    for i in range(number_of_syllables):

        # Randomly choose from initial set
        initial_letter = initial_list[randint(0, len(initial_list)-1)]

        # Randomly choose from medial set
        medial_letter = medial_list[randint(0, len(medial_list)-1)]

        # Randomly choose from terminal set
        terminal_letter = terminal_list[randint(0, len(terminal_list)-1)]

        # Calculate Unicode for syllable block
        syllable_block = (initial_letter * 588) + (medial_letter * 28) + terminal_letter + 44032  

        list_of_syllable_blocks.append(chr(syllable_block))

    return "".join(list_of_syllable_blocks)


def create_many_words_KO(set_of_compatibility_jamo, number_of_words, max_word_length):
    # Lookup lists of Unicode refs for initial, medial, and terminal
    initial_jamo_list, medial_jamo_list, terminal_jamo_list = create_jamo_lists(set_of_compatibility_jamo)

    # Throw error if no initial or medial jamo exist
    if not (initial_jamo_list and medial_jamo_list):
        raise ValueError(f"There must be at least one initial letter and at least one vowel.")

    # Create a list of words
    output = []
    for i in range(number_of_words):
        output.append(create_one_word_KO(initial_jamo_list, medial_jamo_list, terminal_jamo_list, randint(1,max_word_length)))

    return output
```
[View this as a GitHub Gist](https://gist.github.com/colinjroberts/6ff78fdced21f389cf38b3dc58a8feb0)

The resulting Korean text definitely looks odd (there are a lot more 3 and 4 character syllables than I am used to seeing), but it works and that is the expected behavior, so I'm counting it as a success! With that, this first post in this project is complete. I have a simple web app that can accept get requests with English or Korean letters and generate random strings using only the letters provided. 

There are a few ideas floating around in my head for expansion and refinement:
- Should I try to auto-detect language or make that part of the GET request?
- Should only real words be used rather than made up ones? Or maybe real words with some letters replaced?
- Perhaps I could pull from some existing lists of most common words and also throw in some made up ones?
- For Korean, should letters be allowed to be shown even if they aren't perfectly correct grammar, but are commonly used (e.g. ㅋㅋㅋ)?
- For Korean, should I weight syllable block production more toward 2-letter blocks?
- For production, should I try to filter out unsavory words?
- What's a good approach for storing global lookup variables in a Flask app (e.g. if I were running an API with these word generators, it seems worth having the lookup tables ready in memory)?

As for next steps I'll be looking into:
- Design mockups for a front end
- Mock up pages with Flask
- Connect an input field or selection boxes of some kind with the functions above
- Add interactive elements (like text that changes color as one types)

## References found for later use
- [Digital Ocean - Make an app with Flask](https://www.digitalocean.com/community/tutorials/how-to-make-a-web-application-using-flask-in-python-3)