---
title: "Projects"
layout: splash
permalink: /projects/
# date: 2024-06-03T13:30:00-04:00
header:
  overlay_color: "#BA0C2F"
  overlay_filter: "0.2"
  actions:
    - label: "My GitHub"
      url: "https://github.com/raydemay"
      target: _blank
    - label: "LinkedIn"
      url: "https://www.linkedin.com/in/raymond-demay"
      target: _blank
excerpt: "This is a portfolio of my projects"
feature_row:
  - image_path: /assets/images/BuckeyeCurrent_Bike_1Line.jpg
    alt: "Buckeye Current Logo"
    title: "Buckeye Current RW-5"
    url: "https://org.osu.edu/buckeyecurrent/rw-5/"
    target: _blank
    btn_label: "Read More"
    btn_class: "btn--primary"
    excerpt: "RW-5 is attempting to be the first 150kg-class electric motorcycle to achieve an average speed of over 200mph. I was part of the aerodynamics design and manufacturing. I ran CFD analysis in Fluent and assisted in the modeling of the fairings in Solidworks."
feature_row_2:
  - image_path: /assets/images/rpi-adxl-breadboard.jpg
    alt: "Image of Raspberry Pi wired to breadboard containing an accelerometer"
    title: "rpi-adxl345"
    url: "https://github.com/raydemay/rpi-adxl345"
    target: _blank
    btn_label: "Read More"
    btn_class: "btn--primary"
    excerpt: "This Python script was used for data collection in my Experimental Projects capstone"
feature_row_3:
  - image_path: /assets/images/j2_example.png
    image_caption: "Example image of an orbit around Earth changing due to J2 perturbations over one year"
    alt: "Image of Earth with perturbed orbits over one year generated in Python with matplotlib"
    title: "aerospace-tools"
    url: "https://github.com/raydemay/aerospace-tools"
    target: _blank
    btn_label: "Read More"
    btn_class: "btn--primary"
    excerpt: "Repository of some Python and MATLAB scripts that I wrote to either make my life easier in undergrad or just as a fun exercise. The compressible flow scripts are intended to be used on a Ti-84 Plus CE-T Python Edition calculator as an offline version of this online [Compressible Aerodynamics Calculator](https://www.engapplets.vt.edu/fluids/compresssibleAero/calc.html)"
---

{% include feature_row id="feature_row" type ="left" %}
{% include feature_row id="feature_row_2" type ="left" %}
{% include feature_row id="feature_row_3" type ="left" %}
