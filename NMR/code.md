---
title: Procedure for spherical NMR build
authors:
  - name: Leeor Alon, Sipan Hovsepian
exports:
  - format: typst
    template: lapreprint-typst
    output: exports/coil.pdf
kernelspec:
  name: python3
  display_name: Python 3
  language: python
---


# Simulation 

Simulation part explores Python simulation of the magnet. We will start by exploring individual functions, then we would combine them.

## Python code

### Defining Sensor points 
Magpylib measures magnetic field using `Sensor` object. To get measurements from various points, we need to setup those `Sensor` points optimally across a sphere. 
Mathematical algorithm used to achieve optimal placement of points along sphere is Fibonacci lattice.
As we are not only interested in single, outermost sphere, we will need to create multiple concentric spheres embedded into the initial one to "host" the measurement points inside. 

The following code takes the following inputs:
* **`num_pts`** $\rightarrow$ Baseline number of points used to calculate the density of the outermost shell.
* **`r`** $\rightarrow$ Outer radius of the sphere (Maximum radius).
* **`num_r_points`** $\rightarrow$ The number of concentric shells to be generated.
* **`base_coord`** $\rightarrow$ An $[x, y, z]$ array representing the center point (offset) of the sphere.

And returns
* **`Sensor`**  $\rightarrow$ Convenient container for storing 3D position data

```python
import numpy as np
import magpylib as magpy

r_roi=5                     #region of interest
num_pts=100                # baseline number of points on outermost shell
r=r_roi/1000
num_r_points=10             #number of concentric shells
base_coord=[0,0,0]


r_points = np.linspace(0.1/1000,r,num_r_points)    # array where every element is radius of corresponding concentric sphere


for rr in r_points:

    num_pts_each_rad = np.ceil(num_pts*rr / r) #normalizing number of points for each radius based on the ratio of the radius
    indices = np.arange(0, num_pts_each_rad, dtype=float) + 0.5
    
    phi = np.arccos(1 - 2*indices/num_pts_each_rad)
    theta = np.pi * (1 + 5**0.5) * indices
    x, y, z = rr*np.cos(theta) * np.sin(phi), rr*np.sin(theta) * np.sin(phi), rr*np.cos(phi);
    
    arr = np.stack((x,y,z),axis=1)
    
    
    if(rr==0.1/1000):
        arr_full = arr
    else:
        arr_full = np.vstack([arr_full, arr])
    
arr_full[:,0] = arr_full[:,0]+base_coord[0]
arr_full[:,1] = arr_full[:,1]+base_coord[1]
arr_full[:,2] = arr_full[:,2]+base_coord[2]

sensor = magpy.Sensor(position=arr_full,style_size=2)
```

The following illustration helps to visualize the process. 

<img src="./images/define_sensor_points_on_filled_sphere.gif" width="400px" style="display:block; margin:auto;" />



### Cost function

The purpose of Cost function is to calculate field inhomogeneity (in ppm) and mean magnetic field. 

The mathematical formulation is:
$$
\eta = \left| \frac{B_{max} - B_{min}}{B_{mean}} \right| \times 10^6
$$


 ```Python
 def cost_func(B,component=0):
    # Bcomponent=0 for X, Bcomponent=1 for Y and Bcomponent=2 for Z
    Bcomp_flat = B[:,component]   
    # Bcomp_flat = Bcomp.flatten()
    meanfield = np.mean(B[:,component]) 
    eta = 1e6*np.abs((np.max(Bcomp_flat)-np.min(Bcomp_flat))/(meanfield))  
    return eta, meanfield 
 ```

### Calculating and storing magnetic field values in 3D

The following code:

 ```Python
def extract_3Dfields(mags,xmin=-5,xmax=5,ymin=-5,ymax=5,zmin=-5, zmax=5, numberpoints_per_ax = 51,filename='output.pkl',plotting=True,Bcomponent=0,stdforplotting=3):
    
    xs = np.linspace(xmin, xmax, numberpoints_per_ax)
    ys = np.linspace(ymin, ymax, numberpoints_per_ax)
    zs = np.linspace(zmin,zmax,numberpoints_per_ax)
    
    x_mesh, y_mesh, z_mesh=np.meshgrid(xs,ys,zs)    #combines 1D arrays to 3D grid)
        
    B = np.zeros((xs.shape[0],ys.shape[0],zs.shape[0],3))  
    
    for kk in range(zs.shape[0]):
        print('extracting slice:' + str(kk+1) + ' out of:' + str(zs.shape[0]))

        grid_xy = np.array([[(x,y,zs[kk]) for x in xs] for y in ys])

        B[:,:,kk,:] = mags.getB(grid_xy)
                    
    data = {'Bfield': B,'xmin': xmin, 'xmax': xmax, 'ymin': ymin, 'ymax': ymax, 'zmin': zmin, 'zmax': zmax, 'numberpoints_per_ax': numberpoints_per_ax, 'filename': filename, 'coordinates':coordinates}
    
    if(filename != None):
        fileObj = open(filename, 'wb')
        pickle.dump(data,fileObj)
        fileObj.close()
    
    if(plotting==True):
        B_mT = B*1000
        
        # minmin = np.min(Bcomponent.flatten())
        # maxmax = np.max(Bcomponent.flatten())
        meanmean = np.mean(B_mT[:,:,:,Bcomponent].flatten())
        stdstd = np.std(B_mT[:,:,:,Bcomponent].flatten())

        fig, (ax1, ax2, ax3) = plt.subplots(3)
        im1=ax1.imshow(B_mT[:,:,int(zs.shape[0]/2),Bcomponent], vmin=meanmean-stdforplotting*stdstd, vmax=meanmean+stdforplotting*stdstd, cmap='jet', aspect='equal', extent=[xmin, xmax, ymin, ymax])
        add_colorbar(im1)
        ax1.title.set_text('xy plane')
        im2=ax2.imshow(B_mT[:,int(ys.shape[0]/2),:,Bcomponent], vmin=meanmean-stdforplotting*stdstd, vmax=meanmean+stdforplotting*stdstd, cmap='jet', aspect='equal', extent=[xmin, xmax, zmin, zmax])
        add_colorbar(im2)
        ax2.title.set_text('xz plane')
        im3=ax3.imshow(B_mT[int(xs.shape[0]/2),:,:,Bcomponent], vmin=meanmean-stdforplotting*stdstd, vmax=meanmean+stdforplotting*stdstd, cmap='jet', aspect='equal', extent=[ymin, ymax, zmin, zmax])
        add_colorbar(im3)
        ax3.title.set_text('yz plane')
        
        fig.suptitle('B field (mT). Mean: ' + str(round(meanmean,6)) + ', std: '  + str(round(stdstd,3)) )
        fig.show()
        
    return data
 ```


### Refining the colorbar

**Problem:** Matplotlib's default `plt.colorbar()` often generates a colorbar that does not match the height of the main plot, resulting in misaligned layouts. 

**Solution:** The `add_colorbar` function uses `make_axes_locatable` to partition the existing plot axis. It dynamically carves out a dedicated sub-axis (`cax`) 
explicitly for the colorbar, ensuring the height of the colorbar scales perfectly with the parent plot. It then restores the original axis state so subsequent plotting 
commands are unaffected.

```Python
def add_colorbar(mappable):
    from mpl_toolkits.axes_grid1 import make_axes_locatable
    import matplotlib.pyplot as plt
    last_axes = plt.gca()
    ax = mappable.axes
    fig = ax.figure
    divider = make_axes_locatable(ax)
    cax = divider.append_axes("right", size="5%", pad=0.05)
    cbar = fig.colorbar(mappable, cax=cax)
    plt.sca(last_axes)
    return cbar
```


### Creating right hand spherical coil


```Python
def right_hand_spherical_coil(R,N,points,plotting=True):

    L = 1.0        # Total normalized path length

    # Parametric variable
    t = np.linspace(0, L, points)
    
    # Spherical coordinate equations
    theta = np.arccos(1 - 2 * t / L)       # Polar angle varies from pi to 0
    phi = 2 * np.pi * N * t / L            # Azimuthal angle for N turns
    
    # Convert to Cartesian coordinates
    x = R * np.sin(theta) * np.cos(phi)
    y = R * np.sin(theta) * np.sin(phi)
    z = R * np.cos(theta)
    if plotting==True:
        # Plotting the helical winding on the sphere
        fig = plt.figure(figsize=(8, 6))
        ax = fig.add_subplot(111, projection='3d')
        ax.plot(x, y, z, color='blue', linewidth=2, label='Helical Winding')
        ax.set_title('Parametric right Helical Winding on a Sphere')
        
        # Plot the wire path
        # Plot the sphere surface for reference
        u, v = np.mgrid[0:2*np.pi:60j, 0:np.pi:30j]
        xs = R * np.sin(v) * np.cos(u)
        ys = R * np.sin(v) * np.sin(u)
        zs = R * np.cos(v)
        ax.plot_surface(xs, ys, zs, color='lightgray', alpha=0.3, linewidth=0)
        
        ax.set_xlabel('X')
        ax.set_ylabel('Y')
        ax.set_zlabel('Z')
        ax.legend()
        ax.set_box_aspect([1,1,1])
        plt.tight_layout()
        plt.show()
        
    return x,y,z
```


#### 1. Normalized Method (Constant Vertical Spacing)

In this approach, the parameter $t$ (from $0$ to length $L$) is mapped using an inverse cosine function:

$$\theta(t) = \arccos\left(1 - \frac{2t}{L}\right)$$
$$\phi(t) = 2\pi N \frac{t}{L}$$

When converting to the Cartesian $z$-coordinate, the cosine and arccosine cancel out:

$$z = R \cos(\theta) \implies z = R\left(1 - \frac{2t}{L}\right)$$

**Result:** Because $z$ is a strictly linear function of $t$, the vertical distance between each winding is perfectly constant.

---

#### 2. Unnormalized Method (Constant Angular Spacing)

In this simpler approach, the parameter $t$ (from $0$ to $\pi$) acts directly as the polar angle:

$$\theta(t) = t$$
$$\phi(t) = N t$$

Converting to Cartesian yields:

$$Z = R \cos(t)$$

**Result:** The vertical position depends on $\cos(t)$. Because cosine changes slowly at the poles and rapidly at the equator, the windings will bunch up tightly at the top and bottom of the sphere.

---


#### Why Normalization Matters

Choosing between constant angular spacing and constant vertical spacing drastically alters the coil's physical and electromagnetic properties:

* **Physical Geometry:** The unnormalized method causes wire to bunch and overlap heavily at the poles, which is often physically impractical to wind. 
* Normalizing the equations distributes the wire evenly across the height of the sphere.
* **Magnetic Field Uniformity:** In physics, spherical coils ability to generate perfectly uniform internal magnetic fields is only achieved if the number of turns 
* per unit of vertical length ($dz$) is constant. The "normalized" approach guarantees this 
* linear $z$-spacing, making it a strict requirement for accurate scientific instrumentation.

:::{figure} ./images/normalization_problem.png
:label: normalization_problem
:width: 100%

Effect of normalization
::: 


### Calculating curve length

The exact arc length of a 3D curve is:

$$L = \int_a^b \sqrt{\left(\frac{dx}{dt}\right)^2 + \left(\frac{dy}{dt}\right)^2 + \left(\frac{dz}{dt}\right)^2} \, dt$$

This function approximates that integral by replacing it with a finite sum:

$$L \approx \sum_{i=1}^{n-1} \sqrt{(x_{i+1}-x_i)^2 + (y_{i+1}-y_i)^2 + (z_{i+1}-z_i)^2}$$


The code for realization is:

```Python

    if points.shape[0] < 2:
        return 0.0  # A curve needs at least two points to have a length

    # Calculate the differences between consecutive points
    diffs = np.diff(points, axis=0)

    # Calculate the Euclidean distance for each segment
    segment_lengths = np.sqrt(np.sum(diffs**2, axis=1))

    # Sum the lengths of all segments to get the total curve length
    total_length = np.sum(segment_lengths)

    return total_length
```


## Simulation results

# Design and Build

design part

## 3d printing

3d printing part

## winding

winding part

# Testing and validation

testing and validation part



```{code-cell} 
import matplotlib.pyplot as plt
import numpy as np
from numpy.linalg import norm
import magpylib as magpy
import pickle
import pyvista as pv
import pandas as pd

print("Success! The Python code is executing.")
```

