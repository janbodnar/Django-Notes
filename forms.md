# Forms

## Plain form


In `forms.py`:

```python
from django import forms

class MessageForm(forms.Form):

    text = forms.CharField(max_length=255)
```

In `views.py`

```python
from django.shortcuts import render
from django.http import HttpRequest, HttpResponse

from . import forms


def home(req: HttpRequest):

    if req.method == 'GET':

        mform = forms.MessageForm()

        return render(req, 'index.html', {'form': mform})

    if req.method == 'POST':

        mform = forms.MessageForm(req.POST)

        if mform.is_valid():

            text = mform.cleaned_data['text']
            msg = f'you entered: {text}'
            return HttpResponse(msg)
```

The `index.html`:

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>

<body>

    <h2>Message form</h2>

    <form method="post">
        {{ form.as_p }}
        {% csrf_token %}
        <input type="submit" value="Submit">
    </form>


</body>

</html>
```
