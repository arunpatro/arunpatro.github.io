## benchmaring rust vs cpp for ray tracing

### Introduction
The graphics class at NYU was a great motivation for me to dive deep into compiled languages like `Cpp` and `Rust`. I implement simple raytracing scenes in cpp (class default) and later in rust by trying to replicate similar code structure albeit a few obv differences like OOP vs Impl-Traits. 

I profile both code to find out the pros/cons of coding in either language. Cpp is the baseline.

### Observations
We observe that 
1. cpp is faster and produces a smaller binary. 
2. rust is easier to debug

### [Case 01]: Shading a sphere in orthographic view
```
Objects: One sphere at (0, 0, 0) with radius 0.8
Camera: Orthographic
Sensor Size: 800x800 pixels
```

First the cpp code, which uses the Eigen library