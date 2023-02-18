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

Cpp executiontime: 
Rust executiontime:

First the cpp code, which uses the `Eigen` library for linear algebra and `xxxx` library for image utils. 
```cpp
// C++ include
#include <iostream>
#include <string>
#include <vector>

// Utilities for the Assignment
#include "utils.h"

// Image writing library
#define STB_IMAGE_WRITE_IMPLEMENTATION  // Do not include this line twice in your project!
#include "stb_image_write.h"

// Shortcut to avoid Eigen:: everywhere, DO NOT USE IN .h
using namespace Eigen;

void raytrace_shading() {
    std::cout << "Simple ray tracer, one sphere with different shading" << std::endl;

    const std::string filename("shading.png");
    MatrixXd R = MatrixXd::Zero(1600, 1600);  // Store the color
    MatrixXd G = MatrixXd::Zero(1600, 1600);  // Store the color
    MatrixXd B = MatrixXd::Zero(1600, 1600);  // Store the color
    MatrixXd A = MatrixXd::Zero(1600, 1600);  // Store the alpha mask

    // The camera is perspective, pointing in the direction -z and covering the unit square (-1,1) in x and y
    Vector3d origin(-1, 1, 1);
    Vector3d x_displacement(2.0 / A.cols(), 0, 0);
    Vector3d y_displacement(0, -2.0 / A.rows(), 0);

    // Single light source
    const Vector3d light_position(-1, 1, 1);
    double ambient = 0.1;
    MatrixXd diffuse = MatrixXd::Zero(1600, 1600);
    MatrixXd specular = MatrixXd::Zero(1600, 1600);

    // Color of the sphere
    Vector3d diffuse_coeffs(1, 0, 1);
    Vector3d specular_coeffs(1, 1, 1);

    const Vector3d sphereOrigin(0, 0, 0);
    const double sphere_radius = 0.5;

    for (unsigned i = 0; i < A.cols(); ++i) {
        for (unsigned j = 0; j < A.rows(); ++j) {
            // Prepare the ray
            Vector3d ray_origin = origin + double(i) * x_displacement + double(j) * y_displacement;
            Vector3d ray_direction = RowVector3d(0, 0, -1);

            // Intersect with the sphere
            // NOTE: this is a special case of a sphere centered in the origin and for orthographic rays aligned with the z axis
            // solve equation to obtain t
            double dA = ray_direction.squaredNorm();
            double dB = 2 * ray_direction.dot((ray_origin - sphereOrigin));
            double dC = (ray_origin - sphereOrigin).squaredNorm() - sphere_radius * sphere_radius;
            double discriminant = dB * dB - 4 * dA * dC;
            double t;

            if (discriminant == 0) {
                // one solution
                t = -dB / (2 * dA);
            } else if (discriminant > 0) {
                // two solutions
                double t1 = (-dB + sqrt(discriminant)) / (2 * dA);
                double t2 = (-dB - sqrt(discriminant)) / (2 * dA);
                t = std::min(t1, t2);
            } else {
                // no solutions
                continue;
            }

            if (t > 0) {
                // The ray hit the sphere, compute the exact intersection point
                Vector3d ray_intersection = ray_origin + t * ray_direction;

                // Compute normal at the intersection point
                Vector3d ray_normal = ray_intersection.normalized();

                // TODO: Add shading parameter here
                Vector3d light_direction = (light_position - ray_intersection).normalized();
                Vector3d bisector_direction = (light_direction - ray_direction).normalized();
                diffuse(i, j) = std::max(0., light_direction.dot(ray_normal));
                specular(i, j) = std::pow(std::max(0., ray_normal.dot(bisector_direction)), 100);

                R(i, j) = ambient + diffuse_coeffs(0) * diffuse(i, j) + specular_coeffs(0) * specular(i, j);
                G(i, j) = ambient + diffuse_coeffs(1) * diffuse(i, j) + specular_coeffs(1) * specular(i, j);
                B(i, j) = ambient + diffuse_coeffs(2) * diffuse(i, j) + specular_coeffs(2) * specular(i, j);

                // Clamp to zero
                R(i, j) = std::max(R(i, j), 0.);
                G(i, j) = std::max(G(i, j), 0.);
                B(i, j) = std::max(B(i, j), 0.);

                // Disable the alpha mask for this pixel
                A(i, j) = 1;
            }
        }
    }

    // Save to png
    write_matrix_to_png(R, G, B, A, filename);
}
```

The code in rust is 
```rust
pub fn shader() {
    println!("This is a simple ray tracer for a shading a sphere in perspective view.");

    // sphere parameters
    let sphere_center = Vector3::new(0.0, 0.0, 0.0);
    let sphere_radius = 0.6;
    let sphere_color = Vector3::new(1.0, 0.0, 1.0);
    let sphere_specular = 1.;

    // camera parameters
    let camera_position = Vector3::new(-1.0, 1.0, 5.0);

    // image sensor parameters
    let image_origin = Vector3::new(-1.0, 1.0, 1.0);
    let image_width = 1600usize;
    let image_height = 1600usize;
    let x_step = Vector3::new(2.0 / image_width as f32, 0.0, 0.0);
    let y_step = Vector3::new(0.0, -2.0 / image_height as f32, 0.0);
    let mut r_channel: DMatrix<f32> = DMatrix::zeros(image_width, image_height);
    let mut g_channel: DMatrix<f32> = DMatrix::zeros(image_width, image_height);
    let mut b_channel: DMatrix<f32> = DMatrix::zeros(image_width, image_height);
    let mut alpha: DMatrix<f32> = DMatrix::zeros(image_width, image_height);

    // light source parameters
    let light_position = Vector3::new(-1.0, 1.0, 1.0);
    let ambient_light = 0.1;

    for x in 0..image_width {
        for y in 0..image_height {
            // prepare ray
            let ray_origin = image_origin + x as f32 * x_step + y as f32 * y_step;
            // let ray_direction = (ray_origin - camera_position).normalize();
            let ray_direction = Vector3::new(0., 0., -1.).normalize();

            // solve equation to obtain t
            let a = ray_direction.dot(&ray_direction);
            let b = 2. * ray_direction.dot(&(ray_origin - sphere_center));
            let c = (ray_origin - sphere_center).dot(&(ray_origin - sphere_center))
                - sphere_radius * sphere_radius;
            let discriminant = b * b - 4. * a * c;
            let mut t = 1e9;

            // check if intersection occurs
            if discriminant >= 0. {
                let t1 = (-b + discriminant.sqrt()) / (2. * a);
                let t2 = (-b - discriminant.sqrt()) / (2. * a);
                t = t1.min(t2);
            } else {
                continue;
            }

            if t > 0. {
                let ray_intersection = ray_origin + t * ray_direction;
                let surface_normal = (ray_intersection - sphere_center).normalize();
                let light_direction = (light_position - ray_intersection).normalize();
                let bisector_direction = (light_direction - ray_direction).normalize();

                let diffuse_intensity = light_direction.dot(&surface_normal).max(0.);
                // print!("diffuse_intensity: {}", diffuse_intensity);

                assert!(ray_direction.dot(&surface_normal) <= 0.);
                let specular_intensity = bisector_direction.dot(&surface_normal).max(0.).powf(100.);

                r_channel[(x, y)] = (sphere_color[0] * diffuse_intensity
                    + sphere_specular * specular_intensity
                    + ambient_light)
                    .max(0.);
                g_channel[(x, y)] = (sphere_color[1] * diffuse_intensity
                    + sphere_specular * specular_intensity
                    + ambient_light)
                    .max(0.);
                b_channel[(x, y)] = (sphere_color[2] * diffuse_intensity
                    + sphere_specular * specular_intensity
                    + ambient_light)
                    .max(0.);
                alpha[(x, y)] = 1.;
            }
        }
    }

    // save the image
    let mut imgbuf = image::ImageBuffer::new(image_width as u32, image_height as u32);
    for (x, y, pixel) in imgbuf.enumerate_pixels_mut() {
        let r = (r_channel[(x as usize, y as usize)] * 255.0) as u8;
        let g = (g_channel[(x as usize, y as usize)] * 255.0) as u8;
        let b = (b_channel[(x as usize, y as usize)] * 255.0) as u8;
        let a = (alpha[(x as usize, y as usize)] * 255.0) as u8;
        *pixel = image::Rgba([r, g, b, a]);
    }
    imgbuf.save("sphere_shaded.png").unwrap();
}
```