# ExoPlayer Integration

This tutorial will let you understand basic ExoPlayer integration and also you learn about adding one codec extension `vp9` to it. 


###Getting Started

Create a new Android project.

Add following in app's `build.gradle` file.


```bash
compile 'com.google.android.exoplayer:exoplayer:r2.0.4'
```

In the MainActivity of the application. Add the following code.


> Reference : https://google.github.io/ExoPlayer/guide.html

Implement interfaces for ExoPlayer listeners.

```java

public class MainActivity extends AppCompatActivity implements ExoPlayer.EventListener,
        PlaybackControlView.VisibilityListener {
        
        
        
        }

```

Declare following properties. 


```java

    private static final String TAG = PlayerActivity.class.getSimpleName();
    private static final String MP4_URL = "http://html5demos.com/assets/dizzy.mp4";

    SimpleExoPlayer player;
    SimpleExoPlayerView playerView;

```

Create necessary methods for player initialization.


```java


private void createExoPlayerInstance() {

        Handler mainHandler = new Handler();
        BandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();
        TrackSelection.Factory videoTrackSelectionFactory =
                new AdaptiveVideoTrackSelection.Factory(bandwidthMeter);
        TrackSelector trackSelector =
                new DefaultTrackSelector(mainHandler, videoTrackSelectionFactory);

        LoadControl loadControl = new DefaultLoadControl();

         player =
                ExoPlayerFactory.newSimpleInstance(this, trackSelector, loadControl);

    }

    private void preparePlayer() {

        DefaultBandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();
        DataSource.Factory dataSourceFactory = new DefaultDataSourceFactory(this,
                Util.getUserAgent(this, "yourApplicationName"), bandwidthMeter);
        ExtractorsFactory extractorsFactory = new DefaultExtractorsFactory();
        Uri mp4VideoUri =  Uri.parse(MP4_URL)
                .buildUpon().build();
        MediaSource videoSource = new ExtractorMediaSource(mp4VideoUri,
                dataSourceFactory, extractorsFactory, null, null);


        player.prepare(videoSource);

    }

```

In the `OnCreate` function add folllowing code.

```java

@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_player);
        createExoPlayerInstance();

        playerView = (SimpleExoPlayerView) findViewById(R.id.player_view);
        playerView.setPlayer(player);
        playerView.setControllerVisibilityListener(this);
        playerView.requestFocus();

        preparePlayer();
    }


```


One last thing left is to add `SimpleExoPlayerView` in layout.
In `activity_main.xml` add following view.

```xml
<com.google.android.exoplayer2.ui.SimpleExoPlayerView
        android:id="@+id/player_view"
        android:focusable="true"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>


```

Build and Run the app, you will see following screen. You can play the MP4 content.

![ExoPlayer Sample](images/exoplayer_screen_10.png)



### Completed code 
The source code for the sample app can be found on GitHub project. If you get into some problem, clone the project into a local directory and give it a try:

```bash
$ git clone https://github.com/hassanabidpk/DevFestPlayer.git
```

Move to [next](build_vp9_extension_codec.md) chapter for building VP9 Codec and integrating it with ExoPlayer.




