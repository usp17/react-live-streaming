# React Live Streaming with Video SDK

## 4 Steps to Build React Interactive Live Streaming App using Video SDK
### Tools for building an Interactive Live Streaming App
- VideoSDK.Live's React SDK
- VideoSDK.Live's HLS Composition
- VideoSDK.Live's HLS Streaming

### Step 1: Understanding our React Live Streaming App Functionalities and Project Structure
We will be creating this app for 2 types of users, `Speaker` and `Viewer`.
- Speaker will have all media controls i.e. they can toggle their webcam and mic to share their information to the viewers. Speaker can also start HLS stream so that viewer consume the content.
- Viewer will not have any media controls, they will just watch an VideoSDK HLS Stream, which was started by speaker

#### Pre-requisites before starting to write code for our React Live Streaming App:
- VideoSDK Account. If not, you can [signup](https://www.videosdk.live/signup?utm_source=guestpost&utm_medium=devto&utm_campaign=reactlivestreaming)
- Coding environment for React
- Good understanding of React

After our coding environment is setup, we can now start writing our code, first we will create a new React App using create-react-app, also we will install useful dependencies.

```js
npx create-react-app videosdk-interactive-live-streaming-app

cd videosdk-interactive-live-streaming-app

npm install @videosdk.live/react-sdk react-player hls.js
```


#### Project Structure

We will create 3 screens:
1. Welcome Screen
2. Speaker Screen
3. Viewer Screen

Below is the folder structure of our app.

```
root/
├──node_modules/
├──public/
├──src/
├────screens/
├───────WelcomeScreenContainer.js
├───────speakerScreen/
├──────────MediaControlsContainer.js
├──────────ParticipantsGridContainer.js
├──────────SingleParticipantContainer.js
├──────────SpeakerScreenContainer.js
├──────ViewerScreenContainer.js
├────api.js
├────App.js
├────index.js
```

#### App Container
We will prepare a basic `App.js`, This file will contain all the screens and render all of them conditionally according to the `appData` state changes.

`/src/App.js`

```js
import React, { useState } from "react";
import SpeakerScreenContainer from "./screens/speakerScreen/SpeakerScreenContainer";
import ViewerScreenContainer from "./screens/ViewerScreenContainer";
import WelcomeScreenContainer from "./screens/WelcomeScreenContainer";

const App = () => {
  const [appData, setAppData] = useState({ meetingId: null, mode: null });

  return appData.meetingId ? (
    appData.mode === "CONFERENCE" ? (
      <SpeakerScreenContainer meetingId={appData.meetingId} />
    ) : (
      <ViewerScreenContainer meetingId={appData.meetingId} />
    )
  ) : (
    <WelcomeScreenContainer setAppData={setAppData} />
  );
};

export default App;
```

### Step 2: Welcome Screen of our React Live Streaming App 
Creating a new meeting will require an api call, so we will write some code for that

A temporary auth-token can be fetched from our user dashboard, but in production, we recommend to use an authToken generated by your servers.

Follow this guide to get temporary auth-token from user dashboard.

`/src/api.js`

```js
export const authToken = "temporary-generated-auth-token-goes-here";

export const createNewRoom = async () => {
  const res = await fetch(`https://api.videosdk.live/v2/rooms`, {
    method: "POST",
    headers: {
      authorization: `${authToken}`,
      "Content-Type": "application/json",
    },
  });

  const { roomId } = await res.json();
  return roomId;
};
```
WelcomeScreenContainer will be useful for creating a new meeting by speakers. it will also allow to enter already created meetingId to join the existing session.
`src/screens/WelcomeScreenContainer.js`

```js
import React, { useState } from "react";
import { createNewRoom } from "../api";

const WelcomeScreenContainer = ({ setAppData }) => {
  const [meetingId, setMeetingId] = useState("");

  const createClick = async () => {
    const meetingId = await createNewRoom();

    setAppData({ mode: "CONFERENCE", meetingId });
  };
  const hostClick = () => setAppData({ mode: "CONFERENCE", meetingId });
  const viewerClick = () => setAppData({ mode: "VIEWER", meetingId });

  return (
    <div>
      <button onClick={createClick}>Create new Meeting</button>
      <p>{"\n\nor\n\n"}</p>
      <input
        placeholder="Enter meetingId"
        onChange={(e) => setMeetingId(e.target.value)}
        value={meetingId}
      />
      <p>{"\n\n"}</p>
      <button onClick={hostClick}>Join As Host</button>
      <button onClick={viewerClick}>Join As Viewer</button>
    </div>
  );
};

export default WelcomeScreenContainer;
```
### Step 3: Speaker Screen of our React Live Streaming App
This screen will contain all the media controls and participants grid. First, we will create a name input box for participant who will be joining.
`src/screens/speakerScreen/SpeakerScreenContainer.js`

```js
import { MeetingProvider } from "@videosdk.live/react-sdk";
import React from "react";
import MediaControlsContainer from "./MediaControlsContainer";
import ParticipantsGridContainer from "./ParticipantsGridContainer";

import { authToken } from "../../api";

const SpeakerScreenContainer = ({ meetingId }) => {
  return (
    <MeetingProvider
      token={authToken}
      config={{
        meetingId,
        name: "C.V. Raman",
        micEnabled: true,
        webcamEnabled: true,
      }}
      joinWithoutUserInteraction
    >
      <MediaControlsContainer meetingId={meetingId} />
      <ParticipantsGridContainer />
    </MeetingProvider>
  );
};

export default SpeakerScreenContainer;
```
#### MediaControls
This container will be used for toggling mic and webcam. Also, we will add some code for starting HLS streaming.

```js
import { useMeeting, Constants } from "@videosdk.live/react-sdk";
import React, { useMemo } from "react";

const MediaControlsContainer = () => {
  const { toggleMic, toggleWebcam, startHls, stopHls, hlsState, meetingId } =
    useMeeting();

  const { isHlsStarted, isHlsStopped, isHlsPlayable } = useMemo(
    () => ({
      isHlsStarted: hlsState === Constants.hlsEvents.HLS_STARTED,
      isHlsStopped: hlsState === Constants.hlsEvents.HLS_STOPPED,
      isHlsPlayable: hlsState === Constants.hlsEvents.HLS_PLAYABLE,
    }),
    [hlsState]
  );

  const _handleToggleHls = () => {
    if (isHlsStarted) {
      stopHls();
    } else if (isHlsStopped) {
      startHls({ quality: "high" });
    }
  };

  return (
    <div>
      <p>MeetingId: {meetingId}</p>
      <p>HLS state: {hlsState}</p>
      {isHlsPlayable && <p>Viewers will now be able to watch the stream.</p>}
      <button onClick={toggleMic}>Toggle Mic</button>
      <button onClick={toggleWebcam}>Toggle Webcam</button>
      <button onClick={_handleToggleHls}>
        {isHlsStarted ? "Stop Hls" : "Start Hls"}
      </button>
    </div>
  );
};

export default MediaControlsContainer;
```
#### ParticipantGridContainer
This will get all the joined participants from `useMeeting` hook and render them individually. Here we will be using `SingleParticipantContainer` for rendering a single participant's webcam stream.
`src/screens/speakerScreen/ParticipantsGridContainer.js`

```js
import { useMeeting } from "@videosdk.live/react-sdk";
import React, { useMemo } from "react";
import SingleParticipantContainer from "./SingleParticipantContainer";

const ParticipantsGridContainer = () => {
  const { participants } = useMeeting();

  const participantIds = useMemo(
    () => [...participants.keys()],
    [participants]
  );

  return (
    <div>
      {participantIds.map((participantId) => (
        <SingleParticipantContainer
          {...{ participantId, key: participantId }}
        />
      ))}
    </div>
  );
};

export default ParticipantsGridContainer;
```
#### SingleParticipantContainer
This container will get `participantId` from props and will get webcam streams and other information from `useParticipant` hook.
It will render both Audio and Video streams of the participant whose participantId is provided from props.
`src/screens/speakerScreen/SingleParticipantContainer.js`

```js
import { useParticipant } from "@videosdk.live/react-sdk";
import React, { useEffect, useMemo, useRef } from "react";
import ReactPlayer from "react-player";

const SingleParticipantContainer = ({ participantId }) => {
  const { micOn, micStream, isLocal, displayName, webcamStream, webcamOn } =
    useParticipant(participantId);

  const audioPlayer = useRef();

  const videoStream = useMemo(() => {
    if (webcamOn && webcamStream) {
      const mediaStream = new MediaStream();
      mediaStream.addTrack(webcamStream.track);
      return mediaStream;
    }
  }, [webcamStream, webcamOn]);

  useEffect(() => {
    if (!isLocal && audioPlayer.current && micOn && micStream) {
      const mediaStream = new MediaStream();
      mediaStream.addTrack(micStream.track);

      audioPlayer.current.srcObject = mediaStream;
      audioPlayer.current.play().catch((err) => {
        if (
          err.message ===
          "play() failed because the user didn't interact with the document first. https://goo.gl/xX8pDD"
        ) {
          console.error("audio" + err.message);
        }
      });
    } else {
      audioPlayer.current.srcObject = null;
    }
  }, [micStream, micOn, isLocal, participantId]);

  return (
    <div style={{ height: 200, width: 360, position: "relative" }}>
      <audio autoPlay playsInline controls={false} ref={audioPlayer} />
      <div
        style={{ position: "absolute", background: "#ffffffb3", padding: 8 }}
      >
        <p>Name: {displayName}</p>
        <p>Webcam: {webcamOn ? "on" : "off"}</p>
        <p>Mic: {micOn ? "on" : "off"}</p>
      </div>
      {webcamOn && (
        <ReactPlayer
          playsinline // very very imp prop
          pip={false}
          light={false}
          controls={false}
          muted={true}
          playing={true}
          url={videoStream}
          height={"100%"}
          width={"100%"}
          onError={(err) => {
            console.log(err, "participant video error");
          }}
        />
      )}
    </div>
  );
};

export default SingleParticipantContainer;
```
Our speaker screen is completed, not we can start coding `ViewerScreenContainer`

### Step 4: Viewer Screen of our React Live Streaming App
Viewer screen will be used for viewer participants, they will be watching the HLS stream when speaker starts to stream.
Same as Speaker screen this screen will also have initialization process.
`src/screens/ViewerScreenContainer.js`

```js
import {
  MeetingConsumer,
  Constants,
  MeetingProvider,
  useMeeting,
} from "@videosdk.live/react-sdk";
import React, { useEffect, useMemo, useRef } from "react";
import Hls from "hls.js";
import { authToken } from "../api";

const HLSPlayer = () => {
  const { hlsUrls, hlsState } = useMeeting();

  const playerRef = useRef(null);

  const hlsPlaybackHlsUrl = useMemo(() => hlsUrls.playbackHlsUrl, [hlsUrls]);

  useEffect(() => {
    if (Hls.isSupported()) {
      const hls = new Hls({
        capLevelToPlayerSize: true,
        maxLoadingDelay: 4,
        minAutoBitrate: 0,
        autoStartLoad: true,
        defaultAudioCodec: "mp4a.40.2",
      });

      let player = document.querySelector("#hlsPlayer");

      hls.loadSource(hlsPlaybackHlsUrl);
      hls.attachMedia(player);
    } else {
      if (typeof playerRef.current?.play === "function") {
        playerRef.current.src = hlsPlaybackHlsUrl;
        playerRef.current.play();
      }
    }
  }, [hlsPlaybackHlsUrl, hlsState]);

  return (
    <video
      ref={playerRef}
      id="hlsPlayer"
      autoPlay
      controls
      style={{ width: "70%", height: "70%" }}
      playsInline
      playing
      onError={(err) => console.log(err, "hls video error")}
    ></video>
  );
};

const ViewerScreenContainer = ({ meetingId }) => {
  return (
    <MeetingProvider
      token={authToken}
      config={{ meetingId, name: "C.V. Raman", mode: "VIEWER" }}
      joinWithoutUserInteraction
    >
      <MeetingConsumer>
        {({ hlsState }) =>
          hlsState === Constants.hlsEvents.HLS_PLAYABLE ? (
            <HLSPlayer />
          ) : (
            <p>Waiting for host to start stream...</p>
          )
        }
      </MeetingConsumer>
    </MeetingProvider>
  );
};

export default ViewerScreenContainer;
```
Our ViewerScreen is completed, now we can test our application.
`npm run start`

## Output of our React Interactive Live Streaming App
<a href="http://www.youtube.com/watch?feature=player_embedded&v=FfQZnBH3zMQ
" target="_new"><img src="http://img.youtube.com/vi/FfQZnBH3zMQ/0.jpg" /></a>

![React Live Streaming:](https://www.videosdk.live/_next/image?url=https%3A%2F%2Fassets.videosdk.live%2Fstatic-assets%2Fghost%2F2023%2F04%2FReact-Live-Streaming.gif&w=680&q=75)

Source Code of this app is available in this [GithubRepo](https://github.com/ChintanRajpara/videosdk-interactive-live-streaming-app).

## What Next ?
This was a very basic example of interactive Live Streaming App using Video SDK, you can customize it in your way.
- Add more CSS to make the UI more interactive
- Add Chat using [PubSub](https://docs.videosdk.live/react/api/sdk-reference/use-pubsub)
- Implement [Change Mode](https://docs.videosdk.live/react/api/sdk-reference/use-meeting/methods#changemode), by this we can switch any participant from Viewer to Speaker, or vice versa.
- You can also take reference from our Prebuilt App which is build using VideoSDK's React package. Here is the [Github Repo](https://github.com/videosdk-live/videosdk-rtc-react-prebuilt-ui).

## More React Resources
- [React Video Call Quick Start Docs](https://docs.videosdk.live/react/guide/video-and-audio-calling-api-sdk/quick-start)
- [React Interactive Live Streaming Quick Start Docs](https://docs.videosdk.live/react/guide/video-and-audio-calling-api-sdk/quick-start-ILS)
- [Build a Video Chat App with React Hooks](https://dev.to/video-sdk/build-video-calling-app-using-react-hooks-1a79)
- [Code Samples](https://docs.videosdk.live/code-sample)

## Conclusion
With this, we successfully built the React Interactive Live Streaming app with video SDK. You can always refer to our [documentations](https://docs.videosdk.live/). if you want to add features like chat messaging and screen sharing. If you have any problem with the implementation, Please contact us via our [Discord community](https://discord.gg/Gpmj6eCq5u).

Don't forget to share this article on [Twitter ](https://twitter.com/intent/tweet?url=https://www.videosdk.live/blog/react-interactive-live-streaming%0a%0a&text=Build%20a%20React%20Interactive%20Live%20Streaming%20App%20with%20Video%20SDK%0aby%20Chintan%20Rajpara%0afrom%20@video_sdk%20%0a%0a&hashtags=DeveloperBlog,VideoSDK,AudioVideoApp,VideoCallingCustomSDK,PrebuiltSDK).
