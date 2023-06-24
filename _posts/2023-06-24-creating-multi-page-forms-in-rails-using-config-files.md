---
layout: post
title: "📜📜📜 Creating Multi-Page Forms in Rails (using Config files)"
date: "2023-06-24"
---

At work, we needed a multi-page form to be integrated within one of our workflows.

Initially, we looked into SaaS form builders, to ideally avoid adding any unnecessary complexity. Unfortunately, we managed to find only 2 form builder candidates that met our needs, the first of which was *way* too expensive for the company to be able to consider, and the other hadn't implemented or considered security concerns for its webhook, so both were unusable for us.

So, I was asked to have a go at building two "quick-and-dirty" 20+ page multi-page forms directly in our Rails app.

I agreed, and got stuck in straight away... after all, how hard could it be?

## Chapter 1: The First Attempt

### Duplication, Duplication, Duplication 😬

First, I did some research to see whether multi-page forms in Rails were a common thing, and if so, how to go about building one. I immediately came across [a tutorial on Code With Jason, for doing exactly this](https://www.codewithjason.com/rails-multi-step-forms/). For anyone interested in building multi-page Rails forms, I strongly recommend reading this tutorial, it was incredibly helpful and guided the solution we went with.

Based on this, I decided to follow the approach of focusing on making it work for now, hard-coding and duplicating to start with, and trying not to abstract anything too early (as this could otherwise lead to the wrong abstraction and much higher complexity).

I started quick and simple. For each page of the form, I added:
- A controller, with actions for show, next_page, and previous_page
- A form object (a PORO with ActiveModel mixed in for validations, as seen in the tutorial previously mentioned)
- A view
- A new route

This was relatively low complexity initially, but I was underestimating the fact that **we were planning to implement 2 of these forms**, and that **they were each over 20 pages long**.

After continuing down this path for a week or two, development of the form had started to become much slower, and much more complex and confusing, as we were having to redirect from each controller action to the next controller's action, making it very difficult to keep track of the form's overall flow, and we were duplicating controller logic, view logic, and form object logic at an alarming rate.

Unfortunately, I was too deep into the work at that time, and kept ploughing forwards as a way to suppress my anxiety about the approach I was taking, in the faint hope that somehow things might just get better.

Thankfully, respite was just around the corner.

## Chapter 2: Changing Business Requirements, a Flashback, and a Breakthrough

### Saved by Changing Business Requirements: A Chance to Pause and Rethink 🤔

One day, after logging in for another day of head-down work cracking on with this massive form undertaking, I was informed that the project had been put on hold for the time being due to the business being unsure about the current designs and approach, after they had received some very critical user feedback that suggested a major rethink was required.

At first, this hit me hard:
- Was all of my work being retired before even reaching production?
- Was all of my effort wasted? 😔

Because of this, my motivation and energy had dropped very low, and I took most of the day to come to terms with the news. In the afternoon, I decided to go for a walk for some fresh air, and eventually, thoughts started to stir in my head:
- Was my initial approach really sustainable?
- Had we not put the project on hold, would we actually have been able to deliver these forms?
- Would this have been a high-complexity, high-churn area of the codebase that no-one would ever have wanted to touch, due to being difficult to understand, maintain, or change?
- **Was there a better, cleaner way of doing this?**

### A Flashback: Inspired by Previous Architecture 🏛️

In my first role as a Ruby on Rails Engineer, I worked on an internal ticketing application that had a form builder setup, that allowed teams to create their own custom form questions via an admin UI.

I started to wonder whether creating our own internal form builder would be a much simpler, cleaner, and more reusable solution?


After brainstorming this for the rest of the afternoon and evening, I had drawn up plans for new database tables which could house data about each form, each form page, and each form page element - just like the app I had worked on all those years ago.

However, I still had big concerns, mainly around version control: on that ticketing app, the "form builder" was directly used and managed in production by end users through a self-service model. That app also had zero automated testing, and the user's forms were tested live in prod. We didn't want to take that approach here, we wanted to be able to use version control to test our forms in each environment and deploy them following our SDLC process, via our test and deployment pipeline.

### The Breakthrough
The next morning, I had a quick catch-up with my manager to discuss my latest thoughts, and to request some time to spike and explore my latest ideas. My manager was excited about my ideas, and gave me the go-ahead for day of spiking and exploration.

I shared with her my main concern about how to manage version control, and she shared with me links to some open-source code examples of config-driven forms from UK Government projects, suggesting that a config-based approach might be worth exploring.

This was perfect! Things immediately clicked and became clear in my mind: if implemented correctly, we could end up with some shared controllers, shared controller actions, shared form objects, shared views, shared view partials, and shared routes, and there would be very little new code that we'd have to add whenever we wanted to set up a new form, we'd mainly just have to worry about adding new config files for the form (plus manual testing and automated tests, which full disclaimer, I probably didn't implement brilliantly in this project).

During the final implementation of this, after doing some more brainstorming and reviewing of the terminology used by some of the existing multi-page form builders, it struck me that I could visualise the different form page elements as "components", allowing for simple, DRY, shared view partials.

## Chapter 3: Results and Conclusion

### The Final Form Designs

After racing forwards with this idea with my now-limitless amounts of energy, I ended up with two config-driven, tested, multi-page forms integrated fully within our Rails application with all of the functionality desired.

The config for each form was stored under `config/app/multi_page_forms/`, with:
- 1 YAML file per form page;
- 1 YAML file for the overall form structure.

The form config was loaded into the app via an initializer, using the [psych](https://github.com/ruby/psych) Ruby gem, and stored in a namespaced constant as a frozen OpenStruct, with each key and property within the OpenStruct also frozen.

The form config layout and structure was validated within the initializer via a JSON schema, using the [json-schema](https://github.com/voxpupuli/json-schema) Ruby gem. This gave me a really clean way to validate the schema structure and values wouldn't break the app, as well as to act as documentation for future engineers working on the code to see the available form designs, components and structures that they could use.

(Note: Adding a wiki page detailing the available approaches, or perhaps a README.md file, would likely have also been very helpful.)

Other than the form config, here are the DRY parts of the code from the final solution:
- A shared controller, with shared `show`, `next_page` and `previous_page` actions.
- A shared multi-page form object, with shared validation logic, and shared page navigation logic.
- Shared view files and partials, including:
  - text elements (heading1, heading2, heading3, paragraph).
  - input elements (single-select checkboxes, multi-select checkboxes, text input fields, text area fields, searchable multi-select fields).

Here are the non-DRY parts of the code from the final solution:
- Each form required its own routes, for visiting the form's title page, for submitting the form, and for visiting the form's results page (if applicable).
- Each form required its own controllers for inheriting the shared controller actions (to link up with the relevant routes), and some unique routes for entering the form's flow (i.e. showing the form's title page), for submitting the form, and for visiting the form's results page (if applicable).

Other key details about each form:
- The form used Hotwire Turbo to move forward and back between form pages via JavaScript, without having to reload the web browser page or change URL.
- The form used Stimulus JS for all other front-end JavaScript behaviour.
- The users would only be able to visit the form starting from the title page, or going directly to the results page - everything in between (i.e. individual form pages, and submitting the form) were not exposed outside of the flow.
- Any form data was stored in the user's session storage between pages.
- User form data validations were added via the form config (with underlying shared validation logic defined in the shared form object files).
- Users were prevented from moving forward in the form (including when submitting the form) without passing all validations for the current page, with any error messages shown to the users detailing any issues with their inputs.

Testing:
- The most important testing for this was manual testing.
- The Rails app uses RSpec, so I added system tests for each form using RSpec and Capybara that covered the main user flows, and tested each piece of functionality (I didn't test every single possible edge-case, as the forms were just too big for this, but with lots of the code being reused, this was sufficient).

### Post-implementation

After creating this, I onboarded one of my colleagues into the project, and added image elements to the form page views, in a DRY way, so that the image details simply needed to be added within the YAML config file, and the image would be added into the relevant form page as desired.

### Conclusion and Post-Project Thoughts

I really enjoyed this project, and am really proud of the solution I came up with. It's a clean, reusable setup that met the business requirements and robustly solved the problem. ✨🎉🥳

If I was to make further improvements, it would be to:
1. Add a README file with details of the options for adding new form config.
2. Add advice to the README on what form-specific code needed to be added for a new form to work.
3. Review the automated testing approach of the forms, to see whether there are any patterns for making the current tests more simple, clean, maintainable, and/or changeable.

In an ideal world though, we would've used the main SaaS form builder on the market, to save hundreds of engineering hours, it's just a shame that they are so expensive!
