Adding a CommentBox to posts with React
=======================================
In this workshop we are going to build a comment box for our [Reddit clone](https://github.com/ziad-saab/reddit-clone-frontend-workshop). The comment box will have the following behavior:

1. The user gets to the home page and views a list of posts.
2. The user clicks on a "view comments" link for an interesting post.
3. When clicking on this link, the user is taken to `/content/123` where 123 is the ID of the content
4. Upon loading the page, the user will only see the content information, as well as a button saying "Load Comments".
5. When clicking on "Load Comments", the CommentBox component should load the top 25 comments for the current content in reverse chronological order and display them to the user.
6. Above the list of comments, the user will be given a form. The form contains a `<textarea>` in which the user can enter a comment, and a "Send Comment" button.
7. When clicking on "Send Comment", the CommentBox will do an *ajax* POST request to the server to add the comment
8. The server will reply with success or failure
9. If the server replies with a success, we can add the comment on top of the list
10. If the server replies with an error, we can tell the user using an `alert` for the moment.

## Server-side
On the server side we have two things to do:

1. Create a `POST /createComment` endpoint on your server. This should receive form data with a `contentId` and a `commentText`. *Note: where does the userId come from??*
2. Create a `GET /comments/:contentId` endpoint on your server. This endpoint should return a **JSON encoded** array containing all the comments for that contentId.

## Client-side
In a previous workshop, we saw how to render HTML on the server with React and the JSX syntax. We did this by writing pure functions that take data and return part of a UI as a structure built with JSX. For example:

```javascript
function Post(props) {
  return (
    <div>
      <h2>
        <span>{props.voteScore}</span>
        <a href={props.url}>{props.title}</a>
      </h2>
      <cite>Posted by {props.postedBy}</cite>
    </div>
  );
}
```

Now that we are writing interactive code, we will have to use `React.createClass` to create components that can hold **state** and behave dynamically.

Here is a comparison between a comment box built with jQuery and another one with React:

### jQuery
```javascript
var fakeComments = [
    {
        id: 12,
        text: 'hello world!',
        postedBy: 'mark'
    },
    {
        id: 15,
        text: 'bye bye world!',
        postedBy: 'some_guy'
    }
];

function makeCommentListItem(item) {
    return '<li><p>' + item.text + '</p><p>Posted by: ' + item.postedBy + '</p></li>';
}

var $ = require('jquery');

$(document).ready(function() {
    var $button = $('<button>Load Comments</button>');
    $('.comments').append($button);

    $button.on('click', function() {
        // "ghetto" way of receiving the contentId, but it works :)
        // The pathname will be e.g. /content/123
        var contentId = window.location.pathname.split('/')[2];

        //$.getJSON('/content/' + contentId) we dont' have this endpoint yet, let's fake it!

        var $formHtml = $('<form> <input type="hidden" name="contentId" value="' + contentId + '"> <textarea name="commentText"></textarea> <button type="submit">Go!</button>  </form>');

        var $htmlComments = $('<ul class="commentList">' + fakeComments.map(makeCommentListItem).join('\n') + '</ul>');

        $('.comments').empty().append($formHtml).append($htmlComments);

        $formHtml.on('submit', function(event) {
            event.preventDefault();

            var commentData = $formHtml.serialize();

            $.post('/createComment', commentData).then(
                function(result) {
                    $('.commentList').prepend(
                        makeCommentListItem(result)
                    );
                }
            )
        });
    })
});
```

### React
```javascript
var fakeComments = [
    {
        id: 12,
        text: 'hello world!',
        postedBy: 'mark'
    },
    {
        id: 15,
        text: 'bye bye world!',
        postedBy: 'some_guy'
    }
];

var React = require('react');
var ReactDOM = require('react-dom');
var fetch = require('isomorphic-fetch');

function serialize(data) {
    return Object.keys(data).map(function (keyName) {
        return encodeURIComponent(keyName) + '=' + encodeURIComponent(data[keyName])
    }).join('&');
};

var CommentBox = React.createClass({
    getInitialState: function() {
        return {
            displayed: false,
            comments: [],
            loading: false
        };
    },
    loadComments: function() {
        this.setState({
            displayed: true,
            comments: fakeComments
        });
        
        
    },
    sendComment: function(e) {
        e.preventDefault();
        
        var text = this.refs.textInput.value;
        var id = this.refs.contentInput.value;
        
        var f = require('isomorphic-fetch');
        
        var that = this; // why are we doing this??? IF YOU DO NOT KNOW PLEASE ASK!!
        
        f('/createComment', {
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body: serialize({commentText: text, contentId: id}), // this encodes to text=hello+world&contentId=123
            method: 'POST',
            credentials: 'same-origin' // this will send our cookies
        }).then(
            function(r) {
                // r is the response from the server as a JSON string
                
                return r.json(); // this parses the response as JSON to an object
            }
        ).then(
            function(result) {
                // result is the response from the server as an actual object
                
                // here we can finally add the new comment!!
                
                // WHY ARE WE USING that INSTEAD OF this???
                that.state.comments.unshift({
                    id: result.id,
                    text: result.text,
                    postedBy: result.postedBy
                });
                
                // calling this.setState will make React re-render the component
                that.setState({
                    comments: that.state.comments
                });
            }
        );
    },
    render: function() {
        if (this.state.displayed) {
            var contentId = window.location.pathname.split('/')[2];
            
            var commentList = this.state.comments.map(
                function(comment) {
                    return (
                        <li key={comment.id}>
                            <p>{comment.text}</p>
                            <p>Posted by: {comment.postedBy}</p>
                        </li>
                    )
                }
            )
            
            return (
                <div>
                    <form onSubmit={this.sendComment}>
                        <input ref="contentInput" type="hidden" name="contentId" value={contentId} />
                        <textarea ref="textInput" name="commentText"></textarea>
                        <button type="submit">Go!</button>
                    </form>
                    <ul className="comments-list">
                        {commentList}
                    </ul>
                </div>
            );
        }
        else {
            return (
                <div>
                    <button onClick={this.loadComments}>Load Comments</button>
                </div>
            );            
        }
    }
});

ReactDOM.render(<CommentBox/>, document.getElementById('comments'));
```

### How to AJAX with React?
A question that often arises is *"How do I make AJAX calls with React?"* Well, you don't! **React is a UI library**. It's awesome at creating user interfaces and making them reactive, but it doesn't care about how you fetch your data.

Notice at the top of the React version, we're using a library called `isomorphic-fetch` to achieve this. In the `sendComment` function, we are using the `fetch` library to do our AJAX call. While this may be a bit longer than the jQuery version, a few things...:

1. We can tuck all that functionality in a function and re-use it as much as we need in a user-friendly way
2. The `fetch` API will become part of browsers. Soon we won't even need to load the library to get the function
3. jQuery stinks!? Nah, but it's bulky, and if we're only using it to make AJAX calls, we can do better :)
