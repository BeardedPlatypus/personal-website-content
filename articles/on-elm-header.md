title: Creating a sticky header with elm(-ish)
date: 2018-03-08
type: code
category: programming
tags: elm, html

I have added a simple sticky header to this blog (at the time of writing). I 
figured I would write a little article about the design and code, both as a 
personal reminder and to hopefully help people like you. This sticky header is
written in elm(-ish). For those that do not know [elm](http://elm-lang.org), it
is basically a functional programming languages which compiles into Javascript.
It allows one to build interactive web applications without run-time errors.
Ever since I had to use it for an university project, I have loved this little
language.

Using elm for such a small element of my blog is probably overkill, and a couple
of lines of simple Javascript could have sufficed as well. However, me being me, 
I opted for the more convoluted approach. The main reason to use elm, is to 
explore how well it would play together with 
[Pelican](https://blog.getpelican.com), the static site generator I use for this
blog. Furthermore, since it is such a small feature of the site, it was a good 
simple exercise to keep my elm skills a bit fresh. So without further ado, let's
take a look at the header.

# Layout

The header I had envisioned is quite simple. The top part would be contain 
some branding in the form of "Monthy's Blog" in big letters. Underneath that
the navbar containing links to the blog, about-me and portfolio pages.
The navbar should turn sticky once we scrolled past the branding. 

This leads to the following layout hierarchy: 

* div: banner
    * div: navbar
        * div: brand
        * div: navigation elements
            * link: blog
            * link: about me
            * link: portfolio

with the layout planned, we can move to implementing the header in elm.

# Code and Pelican Interop

We divide the code into two parts, first I'll go over all the elm code needed
to get the header working. Then we will look at the html/javascript side.

## Elm

Web apps, no matter the size, generally use the same typical architecture.
This architecture is explained in depth in the main elm tutorial, which can be
found [here](https://guide.elm-lang.org/architecture/). The basic structure 
consists of three components:

* Model: holds the state of the application
* Msg / Update: provides the method of updating your Model based upon messages
                that the app can send.
* View: describes how the model is displayed and how messages are generated 
        based upon user-input.
        
### Model

```elm
type alias NavElement = (String, String)

type alias Model = { nav_elements : List NavElement
                   , title : String
                   , is_fixed : Bool
                   }
                   
model : List NavElement -> String -> Model
model nav_elements title = { nav_elements = nav_elements
                                  , title        = title
                                  , is_fixed     = False
                                  }
```

The model consists of three elements. `nav_elements` describes the navigation
elements, these consist of a name of the page, i.e. blog, and the url to the
specific website page. The `title` describes the title which is placed in the 
branding section of the header. Finally, `is_fixed` describes whether the 
navigational elements should be fixed or not. 

### Update / Msg

The only change that can currently happen within the header, is the switch 
from non-fixed navigational elements to fixed navigational elements, and 
vice-versa. This is modelled in the `is_fixed` element of the Model, and 
updated in the `update` function. The amount scrolled by the user can be
modelled as a float value. Each time the user scrolls up or down, the app
receives a new float value describing the new situation. As such we need
to define a single message containing this new float:

```elm
type Msg = UpdateScrollPos Float
```

Whenever the user scrolls, this value is passed to the update which function
which determines if the `is_fixed` attribute should be updated.

    :::elm
    update : Msg -> Model -> (Model, Cmd Msg)
    update msg model =
      case msg of
        UpdateScrollPos x ->
          ( { model | is_fixed = x >= 81.3 }
          , Cmd.none
          )

We set `is_fixed` to true, whenever we scroll past 81.3, and to false, when this
is not the case. The value 81.3 is equal to the size of the branding. Of course
it would be significantly more elegant if we could determine this value in code,
instead of hard coding it. However, I have not found this option in elm at the 
current moment. Thus, this has to suffice for now.

### View

Now that we have defined the model, and how to update, it is time to implement 
the determined layout in elm: 

```elm
viewNavElement : NavElement -> Html Msg
viewNavElement (txt, url) =
  li []
     [ a [ href url ]
         [ text txt ]
     ]


viewNavElements : List NavElement -> Bool -> Html Msg
viewNavElements nav_elements is_fixed =
  let
    att = [ class "nav"
          , class "navbar-nav"
          , class "banner-background"
          , class "banner-background-bottom"
          ]
  in
    ul ( if is_fixed then ( class "sticky" ) :: att else att )
       ( List.map viewNavElement nav_elements )
       
view : Model -> Html Msg
view model =
  let
    brand = div ( if model.is_fixed then
                    [ class "navbar-brand"
                    , class "banner-background"
                    , class "fixed-padding"
                    ]
                  else
                    [ class "navbar-brand"
                    , class "banner-background"
                    ]
                )
                [ text model.title ]
    navbar = viewNavElements model.nav_elements model.is_fixed

  in
    div [ id "banner"
        ]
        [ nav [ id    "site-navigation"
              , class "navbar"
              , attribute "role" "navigation"
              ]
              [ brand
              , navbar
              ]
        ]
```

This function describes both the fixed and non-fixed situations, by adding
additional classes to the elements when the header should be fixed. If the 
header is fixed, then the brand should extend to the size of the navigational
elements as well. This way, the content does not jump upward, due to the height
of the navigational elements being removed from the header. This is done by
adding the `fixed_padding` class to the brand. The bar containing the nav
elements is made sticky by adding the `sticky class` to the `ul` class.
The layout of the elements is equal to the layout described earlier.

## Html / Javascript

In order to get the header to fully work, two more problems need to be dealt 
with:

* Initialisation of the pages and title.
* Updates of the scroll position.

### Initialisation of the elm app

In theory, we could hard code the names and pages in the elm code. However,
this would mean, each time we want to make a small change to the name of the
site, or add / remove pages from the navbar, we would need to edit the elm code
again. Hardly a preferable situation. It would be a lot nicer if this data is 
set in the html code which is generated by Pelican. Fortunate for us, this is 
possible by using a mechanism called [flags](https://guide.elm-lang.org/interop/javascript.html).
Basically, we define a set of values to be used in elm, which are provided in
the html file in which the elm app is embedded:

First we define a `Flags` record, describing the elements which will be passed
to the elm app:

```elm
type alias Flags = { brand : String
                   , pages : List (String, String)
                   }
```

Then we use these flags to create an init function, which will be used by the
`Html.programWithFlags` to create our app:

```elm
init : Flags -> (Model, Cmd Msg)
init flags =
    ( (model flags.pages flags.brand)
    , Cmd.none
    )
```

Then finally we can add the appropriate values in the html code:

```javascript
  var node = document.getElementById('header');
  var app = Elm.Header.embed(node, {
    brand: "Monthy's Blog",
    pages: [ ['Blog', '{{ SITEURL  }}/index.html']
           {% for p in pages %}
           , ['{ p.title }}', '{{ SITEURL }}/{{ p.url }}']
           {% endfor %}
           ]
  });
```

Here we see how pelican and elm can play nicely together. The elm app receives
the Flags field as a dictionary parameter. The flags entries contain the values
as they are defined in the elm file, in our case this is brand and pages. 
The pages value is set as a pelican template command, which upon generation will
add all the pages as values in our final generated html code. Success!

### Scroll Updates

Earlier, we mentioned that we receive scroll updates in the form of floats, 
however, I did not specify where these updates came from. Unfortunately, I 
haven't found a way to subscribe to scroll updates directly in elm, so we 
will have to parse them from javascript. This is possible through the use
of ports, which allows javascript to communicate with our elm app. 

In order to use ports we need to define our module as a ports module by adding
the ports keyword in front of module.

```elm
port module Header exposing ( .. )
```

Afterwards we can define our port that will receive the javascript updates:

```elm
port onScroll : (Float -> msg) -> Sub msg

subscriptions : Model -> Sub Msg
subscriptions model = onScroll UpdateScrollPos
```

This port is combined in the subscriptions used by our application. Every time
we receive a float on this port, it is turned into an `UpdateScrollPos` message,
which will be run through our `update` function.

Now that we have defined how to deal with updates on the elm side, let's 
generate them at the javascript side:

```javascript
  window.onscroll = function() {updateScroll()};
  
  function updateScroll() {
    app.ports.onScroll.send(window.pageYOffset);
  }
```

We register a callback on `window.onscroll`. This means that whenever the user
scrolls, our callback is executed. In our case, this executes the 
`updateScroll()` function, which sends the `window.pageYOffset` to our header.
And with that, we now generate updates whenever the user scrolls on the blog,
allowing elm to determine whether the header should be sticky or not.

# Conclusion

In this article I have described how I implemented a simple sticky header in 
elm. Overall I would argue that this approach is probably overkill, and a simple
javascript approach would be both simpler and more performant At the same time,
I have shown that elm can work quite nicely with Pelican, thus proving it feasible
to build apps combining both technologies. With that in mind I am happy with the 
results and will continue using this header for the time being! 

There are two aspects, I will most likely work a bit more on. Firstly, I am not
quite satisfied with the transition of non-sticky to sticky, which seems to be 
primarily caused by the size of the brand padding in the sticky situation. I 
will probably tweak the margin / padding a bit more, to ensure it feels smooth.
Secondly, I would like to add a simple gif/animation to the header. This would 
mean I would need to expand my elm code a little bit to make this possible. 
As such, future versions of the header might look a bit differently than the 
one described here. 

The full code for these examples can be found in 
[this gist](https://gist.github.com/BeardedPlatypus/f33e41a5af8007a947bbcccef50eea1b)

Thanks for reading, and hopefully see you again!
