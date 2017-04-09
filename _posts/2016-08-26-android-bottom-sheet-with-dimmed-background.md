---
published: true
layout: post
date: '2016-08-26 21:06 +0200'
title: Android Bottom Sheet with Dimmed Background
categories:
  - android
---
So, I was developing a new budget feature for Monitor for EnergyHive and Engage and needed a way to display the data without obstructing the current user interface. 

I am going to do a complete redesign of the app sometime soon, but for now, this is how it looks:

![device-monitor-current.png]({{site.baseurl}}/assets/device-monitor-current.png){: .screenshot}

As you can see, the design I have used doesn't leave much space for extra features. I'm in desperate need of a good UI designer.

---

Anyway. As always, the [Google Material Design Spec][materialdesignspec] is a godsend for us developers who have no sense of style. I wanted to achieve something like the first image on the [bottom sheet spec][bottomsheetspec] with nice animations.

[materialdesignspec]: https://material.google.com
[bottomsheetspec]: https://material.google.com/components/bottom-sheets.html

![Bottom Sheet Basic Design]({{site.baseurl}}/assets/material_design_spec_components_bottom_sheets.png){: .screenshot}

Essentially, a modal bottom sheet that contains normal interactive elements rather then a list. I was glad to see that official bottom sheet support was added to the [Android Support Library 23.2][androidsupportlibrary232] but after looking further into the implementation, what I wanted didn't look as simple as I thought it would be.

[androidsupportlibrary232]: http://android-developers.blogspot.it/2016/02/android-support-library-232.html


You see, the support library offered two options - the ability to create a bottom sheet out of any view inside a `CoordinatorLayout` & a `BottomSheetDialog` / `BottomSheetDialogFragment`.

At a glance, the `BottomSheetDialogFragment` is just what I wanted as it hosts a fragment in a bottom sheet, and dims the existing activity. But nope, instead of a smooth upward animation, I got a jarring activity transition exactly like the native sharing "animation" in stock android.

Ugly!

I started playing with `FrameLayouts`, and `TouchEvents` to try and get the animation you see below. I'll go through the steps I went through, and explain briefly. 

![Bottom Sheet Gif]({{site.baseurl}}/assets/bottomsheethires.gif){: .screenshot}

## Creating the Fragment


```java
public class InfoBottomSheetFragment extends Fragment implements LoaderManager.LoaderCallbacks<CostMonthForecast>, View.OnClickListener {

    public InfoBottomSheetFragment() {
        // Required empty public constructor
    }


    public static InfoBottomSheetFragment newInstance(String param1, String param2) {
        InfoBottomSheetFragment fragment = new InfoBottomSheetFragment();
        return fragment;
    }
    
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        View rootView = inflater.inflate(R.layout.fragment_info_bottom_sheet, container, false);
        
        // View references
        
        return rootView;
    }
    
    @Override
    public Loader<CostMonthForecast> onCreateLoader(int id, Bundle args) {
    	return new BudgetLoader(
                getActivity()
        	);
    }
    
    @Override
    public void onLoadFinished(Loader<CostMonthForecast> loader, CostMonthForecast data) {
    	// Update UI
    }
    
    public void setVisibility(boolean visible) {
        isVisible = visible;
        if (visible) {
            if (!budgetLoaderInit || error) {
                getLoaderManager().restartLoader(BUDGET_LOADER, null, this);
                budgetLoaderInit = true;
            }
        }
    }
}
```

I have ommitted most of the code, as it is unnecessary. I tend to use custom `AsyncTaskLoaders` to fetch data from the server. Here, I have inflated the views from the associated layout and created a public method that will allow me to communicate visibility from the `MainActivity` to the `Fragment`.

This may seem all very random but trust me, it will make sense later on.

## Adding the Bottom Sheet Fragment to the Main Activity


```xml
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_activity_coordinator_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <FrameLayout
        android:id="@+id/main_activity_fragment_scrim"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:foreground="@drawable/shape_window_dim"
        >

        <fragment xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:app="http://schemas.android.com/apk/res-auto"
            xmlns:tools="http://schemas.android.com/tools"
            android:name="com.danielstone.energyhive.MainActivityFragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            tools:layout="@layout/fragment_main"
            android:id="@+id/main_activity_fragment" />
    </FrameLayout>

	<!-- The NestedScrollView is the BottomSheet -->

    <android.support.v4.widget.NestedScrollView
        android:id="@+id/bottom_sheet"
        android:layout_width="match_parent"
        android:layout_height="@dimen/budget_fragment_height"
        android:elevation="16dp"
        android:clipToPadding="true"
        android:background="@color/colorWhite"
        app:layout_behavior="android.support.design.widget.BottomSheetBehavior"
        >

        <fragment xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:app="http://schemas.android.com/apk/res-auto"
            xmlns:tools="http://schemas.android.com/tools"
            android:name="com.danielstone.energyhive.InfoBottomSheetFragment"
            android:layout_width="match_parent"
            android:layout_height="@dimen/budget_fragment_height"
            tools:layout="@layout/fragment_main"
            android:id="@+id/info_bottom_sheet_fragment" />

    </android.support.v4.widget.NestedScrollView>

</android.support.design.widget.CoordinatorLayout>
```

As you can see above, I wrapped the main fragment which contains the app (you could wrap your views) in a `FrameLayout` with it's `foreground` set to a drawable that is simply a black rectangle.


### shape_window_dim.xml:  
```xml
<shape
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle" >
    <solid android:color="#000000" />
</shape>
```

So that's the layouts done. Awesome. Now we need to build in the functionality. Here is a code snippet from my MainActivity.java. I'll explain it through comments.

```java

// get the Primary Dark colour as the default status bar colour

final int statusBarColor = ContextCompat.getColor(this, R.color.colorPrimaryDark);

// only apply this if on Android Lollipop or above (KitKat and below don't support this

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    Window window = this.getWindow();

    // clear FLAG_TRANSLUCENT_STATUS flag:
    window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);

    // add FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS flag to the window
    window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);

    // finally change the color
    window.setStatusBarColor(statusBarColor);
}

// Now we need to add BottomSheetBehaviour to the BottomSheet View (in my case the NestedScrollView

mBottomSheetBehavior = BottomSheetBehavior.from(mBottomSheet);

// We have a variable that stores the bottom sheet offset. This will be 1 if up and 0f if down.

bottomSheetSlideOffset = 0f;
if (savedInstanceState != null)
    bottomSheetSlideOffset = savedInstanceState.getFloat("bottomSheetSlideOffset", 0f);
    
// Use my setScrim method passing in the offset and the default statusBarColor

setScrim(bottomSheetSlideOffset, statusBarColor);

// Let's start listening to interaction with the BottomSheet

mBottomSheetBehavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {

	// This method is called when the state changes to one of the constants in BottomSheetBehaviour
    @Override
    public void onStateChanged(@NonNull View bottomSheet, int newState) {
        setBottomSheetState(newState);
		
        // Communicate the state change to the Fragment. I should probably check with instanceof. Adding that to my to-do list.

        InfoBottomSheetFragment infoBottomSheetFragment = (InfoBottomSheetFragment) getSupportFragmentManager().findFragmentById(R.id.info_bottom_sheet_fragment);
        if (newState == BottomSheetBehavior.STATE_SETTLING || newState == BottomSheetBehavior.STATE_EXPANDED) {
        	// Using the public method we made earlier.
            infoBottomSheetFragment.setVisibility(true);
        } else if (newState == BottomSheetBehavior.STATE_COLLAPSED) {
            infoBottomSheetFragment.setVisibility(false);
        }
    }

	// This method is called when the Bottom Sheet moves on the screen.
    @Override
    public void onSlide(@NonNull View bottomSheet, float slideOffset) {
    	// setBottomSheetSlideOffset is a custom method that allows me to change the public variable.
        setBottomSheetSlideOffset(slideOffset);
        
        // Call setScrim every time the offset is changed.
        setScrim(slideOffset, statusBarColor);
    }
});
```

### setScrim()
```java
    private void setScrim(float slideOffset, int statusBarColor) {
        mScrim.getForeground().setAlpha((int) (slideOffset * 150));
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            Window window = MainActivity.this.getWindow();
            // finally change the color
            window.setStatusBarColor(Utility.darker(statusBarColor, 1.3f - slideOffset));
        }
    }
```

So, let's go through this step by step.

1. I'm grabbing the the Primary Dark colour to use as the status bar background. I am only applying this on Android Lollipop and above. Over 70% of my users are on API 21 and above, so I haven't bothered to check before grabbing the colour.
2. Call the setScrim() method. _I am calling it a scrim as that's what Google calls them, although it seems to be a North American word._ Anyway. This method takes the FrameLayout, and cahnges the alpha of the foreground drawable. As the slide offset is between 0 and 1, we can multiply this to get an alpha value - 0 to 255. By changing the factor, we can change the darkest it will become - I have chosen 150, which is about 60% transparency.
3. I also have a Utility method which darkens a colour by a value.

This post is still a work in progress so stay tuned for more. I'm not the best at writing and explaining, but I hope to improve over time :D
