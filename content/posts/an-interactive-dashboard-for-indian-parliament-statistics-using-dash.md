+++ 
date = 2019-06-28T20:56:43+02:00
title = "An interactive dashboard for Indian Parliament statistics using Dash"
slug = "an-interactive-dashboard-for-indian-parliament-statistics-using-dash" 
tags = ["data visualization", "plotly", "politics", "dashboard"]
categories = []
description = "Visit https://lok-sabha-review.herokuapp.com and review the performance of your elected representative."
+++

(Cross posted, originally on medium https://medium.com/@saswatachakravarty/74a6dbb35ecc)

![india map](/india.png)

General elections in India are underway! Do you want to review how your elected representative fared in the past term before going to cast your vote? 
Check out my app: https://lok-sabha-review.herokuapp.com/ . You can use it to compare the performance of parties and elected representatives between previous and the current terms, 
and also view there metrics on a map to understand how they vary across the country.

## Dash

There are a lot of impressive data visualization libraries out there in the python ecosystem. What sets apart Dash is that it allows you 
to build a web application that lets users interact with your visualizations — in just few lines of python code. You do not need to know about 
any javascript frameworks, nor do you need to worry about building any APIs.

You define your graph objects in python in the usual way. It is no different the code for a visualization that one would do while working on 
a jupyter notebook environment. The additional step is in defining an update function, which specifies how the graph would change 
when there is a change in the user input. The update function has to be annotated with the ids of the input and the output elements.

So if you are someone with a pandas data frame and a story to tell, I highly recommend you go to https://dash.plot.ly/installation to get started.

To checkout the application code, or if you want to contribute ideas or new datasets go to the project repository https://github.com/saswatac/loksabhareview .

## Data
The data used in the application has been taken from PRS India. PRS tracks the functioning of the Indian Parliament and is an independent policy research institute.

The geojson data for the parliamentary constituencies have been taken from Datameet . The dataset is shared under CC0 1.0 Universal Public Domain Dedication license.
