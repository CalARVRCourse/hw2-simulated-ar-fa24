# HW2: Building Unity Game in Simulated AR

In this homework you will be building a simple AR Game. The exact game you make is up to you and can be very simple (think Board games such as Tic-Tac-Toe). This document will help you set up an unity game in a simulate environment. It will guide you through the steps of: finding planes in a scene, selecting your game board location, and creating basic interaction elements. 

It will be up to you to use what you have learned to make a game. Your game must have an end condition and indicate to the user the result of the game (ie. whether they have won or lost, or what their final score is). 

## Logistics

Please fork this assignment into a private repo. Rename the repo to cs294-137-hw2-YourGitID 

Note, this stack overflow post covers on how to fork into a private repo: https://stackoverflow.com/questions/10065526/github-how-to-make-a-fork-of-public-repository-private

### Deadline

HW2 is due 09/28/2024, 11:59PM Pacific. Both your code and video need to be turned in for your submission to be complete. The root folder of the repositoy should contain a text file with the link to the Video (can be hosted on Youtube or other video sharing sites). 

### Academic honesty
Please do not post code to a public GitHub repository, even after the class is finished, since these HWs will be reused both  in the future.

This HW is to be completed individually. You are welcome to discuss the various parts of the HWs with your classmates, but you must implement the HWs yourself -- you should never look at anyone else's code.

## Deliverables: 

### 1. Video
You will make a 2 minute video showing off the features of your game. The video must include a verbal description of your project and it’s features. You must also include captions corresponding to the audio. This will be an important component of all your homework assignments and your final project so it is best you get this set up early. 


### 2. Code
You will also need to push your project folder to your your private repo. 
Add the following github IDs so that we can access these:

`cac-berkeley` `KaushikKunal`

**Submit a link to your repo and your video on bCourses.** Do not modify your repo after the submission deadline.

## Setting Up Your Project:
Note the assets used in this assignmente was tested on Unity 2022.3.41f1 and ARFoundation 5.1.5. It may work in other versions too.

In Unity Hub create a new 3D Project. First, we will install the required AR packages. For this, go to `Window→Package Manager→Packages: Unity Registry` and install the `AR Foundation` plugin (this will require a restart).

We must also enable XR simulation. To do this, go to `Edit→Project Settings→Project→XR Plug-in Management` and set `XR Simulation` to `✅`.

Download the `SimEnvironments.unitypackage` from Google Drive : https://drive.google.com/file/d/1YTqAhUooaSUgOJpK8lk5lh8S4NQTdyKy/view?usp=sharing

Import the `SimEnvironments.unitypackage` using Assets->Import 
![i1.JPG](/Instructions/i1.JPG)

First delete the main camera that is placed in the scene by default by Unity. We will replace this by it’s AR Equivalent. For this we will need two additional game objects. Right click on the scene hierarchy and add `XR->AR Session` (manages the life cycle of the AR session) and `XR->XR origin` (like the 'camera').


Next, we will create a XR simulation and load a sample simulation scene. Go to `Window→XR→AR Foundation→XR Environment` (you may merge this tab with the others on the original window by dragging it). Search for the "LivingRoomEnv" prefab in the XR Environment dropdown and load this into the scene. This is the virtual environment in which you will spawn your AR game. This contains the scene, as well as any associated AR planes in it.
![i3.png](/Instructions/i3.png)

Once you click the `►` button, you can interact with the simulation. Hold down right click to pan the camera and use WASDEQ (while right clicking) to move.

If you would like to modify the AR camera parameters (such as the initial position, which is the tiny light blue camera in the image above), you may do so by editing the `LivingRoomEnv` XR environment, specifically its `Simulation Environment` component. Here are some example values -

![i2.png](/Instructions/i2.png)

Now we will add the ability to detect planes. Go to the `XR Origin` object in the scene and in the inspection tab, select `Add Component` and in the search box type “AR Plane Manager” and add it. You will notice that the plane prefab field is empty. We will fill this field by creating our own plane prefab. To do this, add a new `XR->AR Default Plane` to the scene hierarchy and drag it into project window (at the bottom middle), thus creating a prefab. You may delete the `AR Default Plane` object from your scene hierarchy once you have a prefab. Drag the prefab into the empty plane prefab field, as shown below.

![i4.png](/Instructions/i4.png)

Now, save the scene. Click Play. You should see planes appearing on flat surfaces of the AR scene! You may also try changing the value of `Detection Mode` of the `AR Plane Manager`.

![i5.png](/Instructions/i5.png)

## Placing Your Game Board:

Go to `GameObject->3D Object->Cube` and name your new cube, “Game Board”. Lets change this into more of a game board shape. Select the Game Board and in the inspector set the scale x,y and z values to 0.6, 0.02, and 0.6 respectively. Unity is set up such that the values of 1 unit in game coordinates corresponds to 1 meter in physical coordinates. Since the cube model is a 1 unit cube, these scale parameters correspond to a game board that is 60cm width and height with a 2cm thickness. 

For now, we will also deactivate our game board so that will not be present at the start of our game. To do this, deselect the checkbox at the very top of the inspector tab for your game board (right above the word “tag”).

![image14.png](/Instructions/image14.png)

Now we need to set up the code that allows the user to choose a position for the game board. Since we don’t know what the area will look like in advance, we will let the user choose where they want to place the game board. To the `XR origin` object, we will add the component `AR Raycast Manager` in the same fashion as `AR Plane Manager`, except we will keep the prefab empty here. Raycasting is how we convert a 2D position on the screen to a 3D position in world space. The `AR Raycast Manager` lets us raycast to the planar regions detected by the AR Plane Manager we created in the previous step. 

Next we will add the script to actually place the game board. To the `XR origin` object, select `Add Component->New Script` and name it "PlaceGameBoard". Double click on the script (in the box next to the word script, not the component itself) to open it in an editor. 

Set up your script as shown below. **For all code is this document, be sure to read the code and comments to understand what the code is doing.**
```C++

using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.XR.ARFoundation;
using UnityEngine.XR.ARSubsystems; // Required for TrackableType

public class PlaceGameBoard : MonoBehaviour
{
    public GameObject gameBoard;

    private ARRaycastManager raycastManager;
    private ARPlaneManager planeManager;
    private bool placed = false;

    void Start()
    {
        raycastManager = GetComponent<ARRaycastManager>();
        planeManager = GetComponent<ARPlaneManager>();

        // Set detection mode to Horizontal
        planeManager.requestedDetectionMode = PlaneDetectionMode.Horizontal;
    }

    void Update()
    {
        if (!placed)
        {
            // Use touch input if available, otherwise use mouse input (for XR simulation on laptop)
            if (IsTouchOrMousePressed(out Vector2 touchPosition))
            {
                // Perform the AR Raycast
                List<ARRaycastHit> hits = new List<ARRaycastHit>();
                if (raycastManager.Raycast(touchPosition, hits, TrackableType.PlaneWithinPolygon))
                {
                    Pose hitPose = hits[0].pose;

                    // Place and activate the game board
                    gameBoard.SetActive(true);
                    gameBoard.transform.position = hitPose.position;
                    gameBoard.transform.rotation = hitPose.rotation;
                    placed = true;

                    // Disable further plane detection
                    planeManager.requestedDetectionMode = PlaneDetectionMode.None;
                    DisableAllPlanes();
                }
            }
        }
    }

    // Check if touch or mouse is pressed and return the position
    private bool IsTouchOrMousePressed(out Vector2 touchPosition)
    {
        if (Input.touchCount > 0 && Input.GetTouch(0).phase == TouchPhase.Began)
        {
            // Use touch input
            touchPosition = Input.GetTouch(0).position;
            return true;
        }
        else if (Input.GetMouseButtonDown(0))
        {
            // Use mouse input
            touchPosition = Input.mousePosition;
            return true;
        }

        touchPosition = default;
        return false;
    }

    private void DisableAllPlanes()
    {
        foreach (var plane in planeManager.trackables)
        {
            plane.gameObject.SetActive(false);
        }
    }

    public void AllowMoveGameBoard()
    {
        placed = false;
        gameBoard.SetActive(false);
        planeManager.requestedDetectionMode = PlaneDetectionMode.Horizontal;
        EnableAllPlanes();
    }

    private void EnableAllPlanes()
    {
        foreach (var plane in planeManager.trackables)
        {
            plane.gameObject.SetActive(true);
        }
    }

    public bool Placed()
    {
        return placed;
    }
}
```
When you return to the unity editor you should see your script component now has a field for `Game Board`. Drag your game board object from your scene hierarchy into this field. 

You will notice we haven’t actually used our function to allow the user to move the game board once it is placed. So let’s implement that. For this we are going to add a button to the screen that calls “AllowMoveGameBoard” when pressed. 

Select `GameObject->UI->Canvas` to add a Canvas to the scene. The Canvas is where we will place all 2D UI elements that are meant to appear attached to the screen. Now create a button `GameObject->UI->Legacy->Button`. It should automatically appear under the canvas in the hierarchy.

![image5.png](/Instructions/image5.png)

First let’s set the button position and size to be more reasonable than the default. Select your button and inside the inspector change the width and height to 160 and 80 respectively. Then click on the icon showing multiple boxes and lines, above the word “Anchors”. This allows you to choose the general position of your button on the screen. Select bottom+center.  Now change the X and Y values of the Pivot field to 0.5 and 0 respectively. Lastly zero out the transform position, as these will have changed as you were modifying the other values to try and keep the button at its original position.

![image11.png](/Instructions/image11.png)

If you build and run now you should see a button present at the bottom of your screen, but clicking it doesn’t actually do anything. 

To change this, in your button inspector, scroll down to the field labeled `On Click`. Press the + selection at the bottom of this field. Where the word “none” appears now, drag the `XR Origin` from your scene hierarchy into this location. Lastly change the “No Function” selection to `PlaceGameBoard->AllowMoveGameBoard()`. If you build and run now, after placing your game board, pressing this button should bring back the place visualization and allow you to move your game board to a new location. 

![i6.png](/Instructions/i6.png)
Lastly let’s change the button label to something more intuitive than “Button”. Expand your Button object in the scene hierarchy and select the Text object that appears below it. Change the text field under “Text (Script)” in the inspector to “Move Board”. Build and Run to see your changes. 

## Making A Simple Interactable Object:

Making interactable objects in AR is fairly easy. The short version is, we just have to check if a raycast from a user’s touch intersects with an interactable object, and call a function from that object. 

So let’s make a couple objects to interact with. Create two new cubes in the scene, rescale them to be 10cm x 10cm x 10cm, and rename them “AR Button 1” and “AR Button 2”. Drag them to be part of the Game Board in your hierarchy. Reactivate your game board so you can see it, and then reposition your buttons to be at two separate locations on your game board. Don’t forget to deactivate your game board again once you do this. 

![image6.png](/Instructions/image6.png)

To make it easy for us to call different functions for each button we will create an abstract class which each button will inherit from. In the project window at the bottom of the screen, `Right Click->Create->C# Script` and name it "OnTouch3D". All you need to place in this script is:

```C++
public interface OnTouch3D
{
    void OnTouch();
}  
```

And then our `AR Button 1` will inherit from this script and implement this function. For a simple interaction we will make the object move up by 10cm when pressed. In `AR Button 1`, `Add Component->New Script` and name it “ARButton1”. Open this script and place the following code:

```C++
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

// Adding OnTouch3D here forces us to implement the 
// OnTouch function, but also allows us to reference this
// object through the OnTouch3D class.
public class ARButton1 : MonoBehaviour, OnTouch3D
{
    // Debouncing is a term from Electrical Engineering referring to 
    // preventing multiple presses of a button due to the physical switch
    // inside the button "bouncing".
    // In CS we use it to mean any action to prevent repeated input. 
    // Here we will simply wait a specified time before letting the button
    // be pressed again.
    // We set this to a public variable so you can easily adjust this in the
    // Unity UI.
    // While this is not so much useful in simulation where we just do click usign a mouse, it is 
    // essential when we dpeloy our app to a phone, where the input is through touch.
    
    public float debounceTime = 0.3f;
    // Stores a counter for the current remaining wait time.
    private float remainingDebounceTime;

    void Start()
    {
        remainingDebounceTime = 0;
    }

    void Update()
    {
        // Time.deltaTime stores the time since the last update.
        // So all we need to do here is subtract this from the remaining
        // time at each update.
        if (remainingDebounceTime > 0)
            remainingDebounceTime -= Time.deltaTime;
    }

    public void OnTouch()
    {
        // If a touch is found and we are not waiting,
        if (remainingDebounceTime <= 0)
        {
            // Move the object up by 10cm and reset the wait counter.
            this.gameObject.transform.Translate(new Vector3(0, 0.1f, 0));
            remainingDebounceTime = debounceTime;
        }
    }
}
```


Next let’s add a tag to our object to make it easy to tell that this object is interactable. Select your AR Button 1 and at the top of the inspector window, expand the “Tag” dropdown list and select “Add Tag”. Select + and add the tag “Interactable”. This tag will show up on the Tag dropdown list from now on. Select this as the tag for AR Button 1.

![image4.png](/Instructions/image4.png)

Now we need to create the script to actually perform the raycasting to this object. In `XR Origin`, `Add Component->New Script` and name it “ARButtonManager”. In this script place the following: 

```C++
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ARButtonManager : MonoBehaviour
{
    private Camera arCamera;
    private PlaceGameBoard placeGameBoard;

    void Start()
    {
        // Here we will grab the AR camera 
        // This camera acts like any other camera in Unity.
        arCamera = FindObjectOfType<ARCamera>().GetComponent<Camera>();
        // We will also need the PlaceGameBoard script to know if
        // the game board exists or not.
        placeGameBoard = GetComponent<PlaceGameBoard>();
    }

    void Update()
    {
        if (placeGameBoard.Placed() && Input.GetMouseButtonDown(0))
        {
            Vector2 touchPosition = Input.mousePosition;
            // Convert the 2d screen point into a ray.
            Ray ray = arCamera.ScreenPointToRay(touchPosition);
            // Check if this hits an object within 100m of the user.
            //RaycastHit hit;
            //if (Physics.Raycast(ray, out hit,100))
            RaycastHit[] hits;
            hits = Physics.RaycastAll(ray, 100.0F);
            for (int i = 0; i < hits.Length; i++)
            {
                // Check that the object is interactable.
                if(hits[i].transform.tag=="Interactable")
                    // Call the OnTouch function.
                    // Note the use of OnTouch3D here lets us
                    // call any class inheriting from OnTouch3D.
                    hits[i].transform.GetComponent<OnTouch3D>().OnTouch();
            }
        }
    }
}
```
Build and run your game. When you place your game board, you should now see that this button moves up when you touch it. 

It is worth noting that that the button does not actually have to be visible for you to interact with this. This is useful if you want the user to be able to interact with empty spaces on the game board, or just want the selectable region to be bigger than the object that is displayed. 

Let’s turn `AR Button 2` into an invisible button. First, don’t forget to add the “Interactable” tag since we are now going to set up interaction for this button. Then disable the checkbox next to the “Mesh Renderer” component. This stops the button from being rendered, making it essentially invisible. 

![image12.png](/Instructions/image12.png)

Since moving an invisible object doesn’t make much sense, let’s instead have this object display a message on the screen. Add a new script to `AR Button 2` and name it “ARButton2”. Open the script and add the following: 

```C++
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class ARButton2 : MonoBehaviour, OnTouch3D
{
    public Text messageText;

    public void OnTouch()
    {
        messageText.gameObject.SetActive(true);
        messageText.text = "Button2Pressed";
    }
}
```

We will then need to create the Text object for this to reference. Add a Text object to the scene `GameObject->UI->Legacy->Text` and rename it “Message Text”. We will leave this object centered, but be sure the (X,Y,Z) positions are all set to 0. Change the width and height to 800 and 200 respectively and change the font size to 100.  As with our Game Board we are going to set this to inactive to start. 

![image13.png](/Instructions/image13.png)

To make this a proper message, let’s also add a script to make the text go inactive again after a specified time. Add a new script component and name it "DisappearingText". Open it and add the following:

```C++
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class DisappearingText : MonoBehaviour
{
    // displayTime will be set to a public float
    // so that you can easily change it in the Unity UI
    public float displayTime = 1;

    private float timeRemaining;

    // Start is called before the first frame update
    void Start()
    {
        timeRemaining = displayTime;
    }

    // Update is called once per frame
    // but only while the object is active
    void Update()
    {
        timeRemaining -= Time.deltaTime;
        if (timeRemaining < 0)
        {
            timeRemaining = displayTime;
            this.gameObject.SetActive(false);
        }
    }
}
```

Finally, select your `AR Button 2` again and drag your `Message Text` object into the Message Text section of your `AR Button 2` script. 

![image2.png](/Instructions/image2.png)

Build and run your game. After placing your game board you should now be able to click on the location where the button would be (if it were visible) and a message should appear on screen. 

## Finish your game:

The rest is up to you! Using these techniques you need to now design your own game that can run in this simulated environment.
