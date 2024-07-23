---
title: "Experiments: Rails-like Form Helpers"
datePublished: Tue Jun 04 2024 03:01:34 GMT+0000 (Coordinated Universal Time)
cuid: clwztcmcd000e0ak21rm48tzp
slug: experiments-rails-like-form-helpers
tags: forms, reactjs, rsc

---

After ReactConf 2024 I was inspired by this tweet:

%[https://x.com/chantastic/status/1791531154212033004] 

In it, [Sam Selikoff](https://twitter.com/samselikoff) says that the JS ecosystem has yet to demonstrate the same kind of powerful forms that have been available in Rails since just about version 1.0 almost 20 years ago. (He mentions Laravel forms as well, but I don't have any experience with those.) He calls out specifically the ability to edit a doctor, patients and their appointments all in one form, without parsing through every field to format them to submit to the server, and/or hand-parsing the response on the server to save the changes to the database.

Being the resident Rails expert on the RedwoodJS team I took it as a challenge to do something similar! Redwood does feature some very helpful form helpers which handle errors from the server automatically, highlighting the field and showing error messages. But you'd still need to create each and every form field and format the data in an `onSubmit` to fit the shape of the GraphQL mutation. But with RSC we're freed from the world of GraphQL mutations and can save the data any way we want.

## Assumptions

Ideally this code would work with [Server Actions](https://react.dev/reference/rsc/server-actions) as part of the upcoming RSC featureset, but Server Actions aren't ready yet in Redwood. I took the next-best step and created a [function](https://redwoodjs.com/docs/serverless-functions) which is sort of blank canvas and emulates fairly closely how Server Actions will work: a function you can send data into and then run any code you'd like, including writing to a database.

While this code will format the form values in a way similar to Rails, saving it to the database using [Prisma](https://www.prisma.io/orm) would be a different matter altogether. To demonstrate the database save we use the experimental [RedwoodRecord ORM](https://redwoodjs.com/docs/redwoodrecord) and save all of the data in one call. (While you can save the attributes on a single model, you can't yet save all of the nested changes as well in a single call, but hopefully soon!).

This exercise is more to show that formatting database field names similar to Rails and then parsing the form data back into objects and arrays is possible. In fact, using these form helpers in your React app should work with a Rails backend out of the box!

## The Format

The secret sauce to Rails form helpers is the formatting of field names so Rails can tell what object structure they should be in. All query string variables and form variables in your body of a POST are turned into a single object named `params` that's available to be read through and manipulated. The [Rails Guides on form helpers](https://guides.rubyonrails.org/form_helpers.html) tells you everything you need to know about this protocol.

| Form Field Name | Object Type | params Object |
| --- | --- | --- |
| `name` | string | `params.name` |
| `name[first]` | object | `params.name.first` |
| `name[]` | array of strings | `params.name` |
| `addresses[][street]` | array of objects | `addresses[0].street` |

Using this we just need to name form fields using the syntax above. Forms are submitted as `x-www-form-urlencoded` mimetype, meaning they are key/value pairs separated with `&` and URL Encoded. For example, the following form fields:

```xml
<form action="/" method="post">
  <input type="text" name="doctor[name]" value="Dr. John Doe" />
  <input type="text" name="doctor[specialty]" value="Cardiology" />
</form>
```

Would be converted into the following string in the body of the request:

```bash
doctor%5Bname%5D=Dr.%20John%20Doe&doctor%5Bspecialty%5D=Cardiology
```

The server then needs to parse that string and turn it into a Javascript object:

```javascript
{
  doctor: {
    name: 'Dr. John Doe',
    specialty: 'Cardiology'
  }
}
```

## The Code

I've got a sample app that contains the form helpers and several example forms of increasing complexity to demonstrate that the various formats are parsed and transformed back into an object on the server:

[https://github.com/cannikin/ultimate-forms](https://github.com/cannikin/ultimate-forms)

To get it up and running:

```bash
git clone https://github.com/cannikin/ultimate-forms ./ultimate-forms
cd ultimate-forms
yarn install
yarn rw dev
```

You'll be presented with a simple form:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717455815542/9ffe0d22-4052-42e5-9190-83286e5cdd8b.png align="center")

Let's take a look at the code that makes the form on the left in `web/src/SimplePage.jsx` (styles removed for clarity):

```javascript
<FormWith url="/.redwood/functions/form" model={doctor}>
  <HiddenField name="id" />

  <Label name="name" />
  <TextField name="name" />

  <Label name="specialty" />
  <TextField name="specialty" />

  <Submit>Save</Submit>
</FormWith>
```

We have a component `<FormWith>` that knows how to take a model (which is an instance of a class like RedwoodRecord that follows the [ActiveRecord](https://en.wikipedia.org/wiki/Active_record_pattern) pattern of making a single instance represent a single row of data from your database) and setup a context that the nested components like `<TextField>` can use to create the proper name for the `<input>` element that's created.

You denote the URL that the form will submit to (which becomes the form's `action`) and the `model` prop which is the instance that the form fields will be representing. In this case we're going to use a simple object that has the properties that the real model instance would have. That object looks like:

```javascript
const doctor = {
  className: 'Doctor',
  id: 10101,
  name: 'Dr. John Doe',
  specialty: 'Cardiology',
}
```

The form helpers use this object to figure what `name` and `specialty` relate to (a parent with a class name of "Doctor") and names the form fields automatically, using the values as the default `value` fields:

```xml
<form action="/.redwood/functions/form" method="post">
  <input type="hidden" name="doctor[id]" value="10101">
  <label for="doctor[name]">Name</label>

  <input type="text" name="doctor[name]" value="Dr. John Doe">
  <label for="doctor[specialty]">Specialty</label>

  <input type="text" name="doctor[specialty]" value="Cardiology">
  <button type="submit">Save</button>
</form>
```

A plain 'ol HTML form! Submit this form and then watch the console: the `/.redwood/functions/form` function just parrots back the data that it received *after* transforming it back into an object:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717456153944/38022977-1782-48d2-bb07-2b4fbf93b1f3.png align="center")

The form on the right demonstrates that you can use these helpers without the pseudo "model" record populating the `<FormWith>`: you simply need to name your inputs properly and the backend will handle the rest.

## Saving Data

This simple form actually *can* be saved by RedwoodRecord as is! If you take a look at `api/src/functions/form/form.js` you'll see the code commented out:

```javascript
const doctor = Doctor.find(params.doctor.id)
doctor.update(params)
```

Yes, that's it! As long as the object is formatted properly, you can save it with one call in RedwoodRecord. Here's the full function, including parsing the params and returning the result:

```javascript
export const handler = async (event, _context) => {
  const params = toParams(event.body)

  const doctor = Doctor.find(params.doctor.id)
  doctor.update(params)

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ params }),
  }
}
```

`toParams()` is where the magic happens to convert the named forms back into an object. Check out `api/src/lib/helpers.js`

## Doctors → Patients → Appointments

There are several more forms of different complexity, but the last one labeled "Complex Form" accomplishes what Sam mentioned in the tweet: edit a doctor, their patients, and those patients' appointments all at once:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717456435733/65ebb570-4901-4100-b454-de35dd440760.png align="center")

It's not the prettiest form in the world, but we're here for the code!

You can see the format of the model on the left and how that translates into the fields at the right. This form uses the `<FieldsFor>` component to change the "context" for the helpers and to know which model they're trying to represent (again, styles and extra HTML removed for clarity):

```javascript
<h2>Patients</h2>

{doctor.patients.map((patient, index) => (
  <FieldsFor model={patient} index={patient.id} key={index}>
    <Label name="name" />
    <TextField name="name" />

    <Label name="dob" />
    <TextField name="dob" />

    {patient.appointments.map((appointment, index) => (
      <FieldsFor model={appointment} index={appointment.id} key={index}>
        <div>
          <Label name="location" label="Appointment Location" />
          <TextField name="location" />
        </div>
        <div>
          <Label name="date" label="Date" />
          <TextField name="date" />
        </div>
      </FieldsFor>
    ))}
  </FieldsFor>
))}
```

So as we loop through any children we use `<FieldsFor>` to say "these following fields are for `doctor.patient` or `doctor.patient.appointment`" This also uses the `index` prop to tell the form helpers to automatically include the child records' `id` as the key in the nested object. When submitting this form, the resulting `params` object looks like this:

```javascript
{
  "params": {
    "doctor": {
      "name": "Dr. John Doe",
      "specialty": "Cardiology",
      "patient": {
        "123": {
          "name": "Jane Doe",
          "dob": "01/01/1980",
          "appointment": {
            "742": {
              "location": "San Diego, CA",
              "date": "01/01/2021"
            }
          }
        },
        "456": {
          "name": "Jeff Generic",
          "dob": "01/01/1970",
          "appointment": {
            "112": {
              "location": "San Francisco, CA",
              "date": "01/02/2021"
            },
            "789": {
              "location": "San Diego, CA",
              "date": "01/01/2021"
            }
          }
        }
      }
    }
  }
}
```

In theory, the code to save this to the database is the same as in the simple case, just `doctor.update(params)`!

The form components and functions that turn objects into the named form elements lives in `web/src/lib/helpers.js` There are plenty of tests for both the web and api helpers.

## What About Types?

The helpers described above takes the stance that these from helpers, while doing some magic behind the scenes, should still behave just like a native HTML form, thus relying on the www-form-urlencoded body. And as far as the HTTP protocol is concerned, everything is a string. You'll notice in the www-form-urlencoded format there are no quotes or anything else that could used as a delimiter to even tell the difference between numbers and text.

In Ruby and Rails this wasn't an issue: Ruby relies on [duck-typing](https://en.wikipedia.org/wiki/Duck_typing) and things would be coerced into the expected type correctly 95% of the time. For the few things that were incorrectly identified you could add a `.to_i` or `.to_boolean` to turn them into real primitives.

But, the helpers could be modified to submit data in another format. Perhaps the form data gets packaged up as JSON and submitted to the server with the `Content-Type: application/json` header. Or use something like [zod.preprocess()](https://zod.dev/?id=preprocess) to coerce values to their expected types once they get to the server.

## What's Next?

We'll be looking at including these form helpers along with RSC and Server Actions in the [Bighorn release of Redwood](https://redwoodjs.com/blog/bighorn-update), sometime later this year. What do you think? Let us know in the [community](https://community.redwoodjs.com/)!