# US73: Feature Data Collection Class Diagram

These diagrams show how the reel-analyzer app collects videos, extracts their feature data with AI, and where that data will be stored. Class and field names match the code as of 11/05/2026 on `master` (video analysis), `US87-YT-Filters-UI` (YouTube search), and `US88-Make-Database-using-Supabase` (database schema).

## Diagram 1: Collecting and analyzing videos

The YouTube search collects candidate videos and their statistics. Video analysis extracts up to 30 frames with ffmpeg and sends them to Claude, which returns structured feature data.

```mermaid
---
title: "US73 Diagram 1: Collecting and analyzing videos"
config:
  theme: neutral
  class:
    hideEmptyMembersBox: true
---
classDiagram
direction LR

namespace Collection_US87 {
    class YouTubeSearch {
        <<component>>
        -search: string
        -videos: Video[]
        -sortBy: string
        -eventType: string
        -videoDuration: string
        -videoDefinition: string
        -topicId: string
        -category: string
        +searchYouTube() void
        +fetchCategories() void
    }
    class YouTubeSearchRoute {
        <<route GET /api/youtube>>
        -apiKey: string
        +GET(request: NextRequest) Response
        -formatVideoData(id: string) Video
    }
    class CategoriesRoute {
        <<route GET /api/ytcategories>>
        -apiKey: string
        +GET(request: NextRequest) Response
    }
    class Topic {
        <<interface>>
        +id: string
        +title: string
    }
    class Video {
        <<interface>>
        +id: string
        +title: string
        +channel: string
        +thumbnail: string
        +views: string or null
        +likes: string or null
        +comments: string or null
    }
    class YouTubeDataAPI {
        <<external>>
        +search.list()
        +videos.list()
        +videoCategories.list()
    }
}

namespace Analysis_master {
    class VideoDescriber {
        <<component>>
        -selected: VideoId
        -status: Status
        +analyze() void
    }
    class AnalyzeVideoRoute {
        <<route POST /api/jshipton>>
        +POST(request: Request) Response
    }
    class VideoCatalog {
        <<module>>
        +VIDEOS: fitness, food
        +isVideoId(value: unknown) boolean
    }
    class FrameExtractor {
        <<module>>
        -MIN_INTERVAL_SECONDS: 2
        -MAX_FRAMES: 30
        -FRAME_WIDTH: 512
        +extractFrames(videoPath: string) ExtractedFrames
        -getDurationSeconds(videoPath: string) number
    }
    class ExtractedFrames {
        <<interface>>
        +durationSeconds: number
        +frames: Frame[]
    }
    class Frame {
        <<interface>>
        +timestamp: number
        +data: string
    }
    class VideoAnalyzer {
        <<module>>
        -MODEL: string
        -PROMPT: string
        +analyzeVideo(frames: ExtractedFrames) VideoAnalysis
    }
    class ClaudeClient {
        <<module>>
        -client: Anthropic
        +getClaudeClient() Anthropic
    }
    class VideoAnalysisError {
        <<exception>>
    }
    class VideoAnalysis {
        <<zod schema>>
        see Diagram 2
    }
    class FFmpeg {
        <<external>>
    }
    class ClaudeAPI {
        <<external>>
        +messages.parse()
    }
}

YouTubeSearch ..> YouTubeSearchRoute : fetch
YouTubeSearch ..> CategoriesRoute : fetch
YouTubeSearch ..> Topic : topic filter
YouTubeSearch "1" o-- "0..50" Video : displays
YouTubeSearchRoute ..> Video : creates
YouTubeSearchRoute ..> YouTubeDataAPI : HTTPS
CategoriesRoute ..> YouTubeDataAPI : HTTPS

VideoDescriber ..> AnalyzeVideoRoute : fetch
AnalyzeVideoRoute ..> VideoCatalog : validates id
AnalyzeVideoRoute ..> FrameExtractor : 1. extractFrames
AnalyzeVideoRoute ..> VideoAnalyzer : 2. analyzeVideo
FrameExtractor ..> FFmpeg : runs
FrameExtractor ..> ExtractedFrames : returns
ExtractedFrames "1" *-- "1..30" Frame
VideoAnalyzer ..> ClaudeClient : uses
ClaudeClient ..> ClaudeAPI : HTTPS
VideoAnalyzer ..> VideoAnalysisError : throws
VideoAnalyzer ..> VideoAnalysis : returns
VideoDescriber ..> VideoAnalysis : displays
```

## Diagram 2: Feature data and where it is stored

`VideoAnalysis` is the feature data Claude returns. The `research_*` tables from the US88 schema store collected videos, their statistics over time, their AI features, and tags. Dotted lines labeled "planned" are designed but not wired up yet: `saveVideo()` is commented out in the YouTube route, and nothing writes to `research_video_features` yet.

```mermaid
---
title: "US73 Diagram 2: Feature data and where it is stored"
config:
  theme: neutral
  class:
    hideEmptyMembersBox: true
---
classDiagram
direction LR

namespace FeatureData_master {
    class VideoAnalysis {
        <<zod schema>>
        +title: string
        +category: string
        +topic: string
        +summary: string
        +hook: string
        +subjects: string[]
        +setting: string
        +onScreenText: string[]
        +keyMoments: KeyMoment[]
        +shots: Shot[]
        +visualStyle: VisualStyle
        +mood: string
        +targetAudience: string
    }
    class KeyMoment {
        +timestamp: number
        +description: string
    }
    class Shot {
        +startTime: number
        +endTime: number
        +shotType: ShotType
        +cameraAngle: CameraAngle
        +cameraMovement: CameraMovement
        +description: string
    }
    class VisualStyle {
        +lighting: string
        +colorPalette: string
        +editingPace: EditingPace
    }
    class ShotType {
        <<enumeration>>
        extreme close-up
        close-up
        medium close-up
        medium
        medium wide
        wide
        extreme wide
        insert/detail
        point of view
    }
    class CameraAngle {
        <<enumeration>>
        eye level
        high angle
        low angle
        overhead
        dutch angle
        over the shoulder
    }
    class CameraMovement {
        <<enumeration>>
        static
        handheld
        pan
        tilt
        tracking
        zoom
        unclear
    }
    class EditingPace {
        <<enumeration>>
        slow
        moderate
        fast
    }
    class Video {
        <<interface, US87>>
        see Diagram 1
    }
}

namespace Storage_US88 {
    class research_channels {
        <<table>>
        +id: varchar PK
        +name: varchar
        +subscribers: bigint
        +queried_at: timestamptz
    }
    class research_videos {
        <<table>>
        +id: varchar PK
        +channel_id: varchar FK
        +title: varchar
        +thumbnail: varchar
        +description: varchar
        +published_at: timestamptz
    }
    class research_video_statistics {
        <<table>>
        +id: bigint PK
        +video_id: varchar FK
        +views: bigint
        +likes: bigint
        +comments: bigint
        +queried_at: timestamptz
    }
    class research_video_features {
        <<table>>
        +id: bigint PK
        +video_id: varchar FK unique
        +ai_description: varchar
    }
    class tags {
        <<table>>
        +id: bigint PK
        +name: varchar unique
        +created_at: timestamptz
    }
    class research_video_tags {
        <<table>>
        +research_video_id: varchar FK
        +tag_id: bigint FK
    }
}

VideoAnalysis "1" *-- "0..*" KeyMoment
VideoAnalysis "1" *-- "0..*" Shot
VideoAnalysis "1" *-- "1" VisualStyle
Shot ..> ShotType
Shot ..> CameraAngle
Shot ..> CameraMovement
VisualStyle ..> EditingPace

research_channels "1" -- "0..*" research_videos : publishes
research_videos "1" -- "0..*" research_video_statistics : snapshot over time
research_videos "1" -- "0..1" research_video_features : AI features
research_videos "1" -- "0..*" research_video_tags
tags "1" -- "0..*" research_video_tags

Video ..> research_videos : planned saveVideo
Video ..> research_video_statistics : planned saveVideo
VideoAnalysis ..> research_video_features : planned save as ai_description
VideoAnalysis ..> tags : planned category and subjects
```

## How to read the diagrams

| Notation | Meaning |
| --- | --- |
| `<<component>>` | React component in the dashboard |
| `<<route ...>>` | Next.js API route, with its HTTP method and path |
| `<<module>>` | TypeScript module of functions and constants (not a class in the code) |
| `<<interface>>`, `<<zod schema>>` | Data shape passed between parts of the app |
| `<<table>>` | Supabase table from `src/lib/supabase/schema.sql` |
| `<<external>>` | Outside service or tool |
| `+` / `-` | Exported (public) / file-private |
| Dotted arrow | Uses, calls, or creates |
| Filled diamond | Composition: the part only exists inside the whole |
| Hollow diamond | Aggregation: the component holds a list of them |
| Plain line with numbers | Database relationship with its multiplicity |

Images of both diagrams are in this folder for viewers that don't display Mermaid.
