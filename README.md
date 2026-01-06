# DeepStream Backend - Step-by-Step Logic Flow

## User Journey with Backend Processing

---

# PHASE 1: APP LAUNCH

## Step 1: User Opens App (Cold Start)

### Client Action:
```
User taps DeepStream icon
App launches
Shows splash screen: "Star Field" fading in
```

### Backend Request #1: Initialize Session
```
POST /api/session/init

Request Body:
{
    device_id: "iPhone_17_Pro_ABC123",
    device_model: "iPhone17,3",
    ios_version: "17.2",
    app_version: "1.0.0",
    user_id: null  // First time user
}

Backend Processing:
1. Generate new session_id (UUID)
2. Check if device_id exists in database
   - IF new device:
       → Create new user record
       → Generate user_id
       → Set first_launch = true
   - IF returning device:
       → Load existing user_id
       → Load user preferences
       → Set first_launch = false

3. Create session record in database:
   INSERT INTO user_sessions (
       session_id,
       user_id, 
       device_id,
       session_start,
       last_active
   )

4. Check for any pending notifications or updates

Response:
{
    session_id: "550e8400-e29b-41d4-a716-446655440000",
    user_id: "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    is_first_launch: true,
    default_station: 99.9,
    greeting: "Welcome to DeepStream"
}

Time: ~50ms
```

---

## Step 2: Load Initial Station (The "Tune" Action)

### Client Action:
```
Star field animation completes
App needs to load first artist station
Show loading indicator
```

### Backend Request #2: Get Station Data
```
GET /api/stations/99.9

Backend Processing:

1. Query Database for Artist:
   SELECT * FROM artists 
   WHERE station_frequency = 99.9
   
   Result: 
   {
       id: "artist_wizkid_001",
       name: "Wizkid",
       station_frequency: 99.9,
       genre: "Afrobeats",
       origin: "Lagos, Nigeria",
       scene_id: "scene_lagos_street"
   }

2. Check Redis Cache:
   KEY: "station:99.9:full"
   
   IF cache HIT:
       → Return cached data (5ms response)
   
   IF cache MISS:
       → Continue to step 3

3. Load Scene Data:
   SELECT * FROM scenes 
   WHERE id = "scene_lagos_street"
   
   Result:
   {
       name: "Lagos Street",
       video_url_high: "cdn.deepstream.io/lagos_street_high.mp4",
       video_url_medium: "cdn.deepstream.io/lagos_street_med.mp4",
       video_url_low: "cdn.deepstream.io/lagos_street_low.mp4",
       particle_config: {
           type: "dust",
           count: 300,
           color: "#FFA500"
       }
   }

4. Select Appropriate Video Quality:
   IF device_model IN ["iPhone17,2", "iPhone17,3", "iPhone16,2", "iPhone16,2","..."]:
       video_url = video_url_high
   ELIF device_model IN ["iPhone13,1", "iPhone13,2"]:
       video_url = video_url_medium
   ELSE:
       video_url = video_url_low

5. Load Influence Orbs:
   SELECT io.*, a.name, a.station_frequency
   FROM influence_orbs io
   JOIN artists a ON io.target_artist_id = a.id
   WHERE io.source_artist_id = "artist_wizkid_001"
   ORDER BY io.orbit_radius ASC
   
   Result: [
       {
           target_artist: "Fela Kuti",
           relationship_type: "ancestor",
           orbit_radius: 150,
           orbit_speed: 0.3,
           angle: 45
       },
       {
           target_artist: "Sade",
           relationship_type: "influence",
           orbit_radius: 120,
           orbit_speed: 0.5,
           angle: 180
       }
   ]

6. Calculate 3D Orb Positions:
   FOR each orb:
       x = orbit_radius * cos(angle_degrees * π/180)
       y = orbit_radius * sin(angle_degrees * π/180) * sin(inclination)
       z = orbit_radius * sin(angle_degrees * π/180) * cos(inclination)
   
   Add to orb_positions array

7. Identify Adjacent Stations (for preloading):
   SELECT station_frequency, id 
   FROM artists 
   WHERE station_frequency IN (98.9, 100.9)
   
   Result: [
       { frequency: 98.9, artist_id: "artist_davido_002" },
       { frequency: 100.9, artist_id: "artist_burna_003" }
   ]

8. Generate Haptic Configuration:
   Based on scene_type = "lagos_street":
   haptic_pattern = {
       lock_pattern: [
           { type: "transient", time: 0, intensity: 1.0, sharpness: 0.5 },
           { type: "continuous", time: 0.05, duration: 0.1, intensity: 0.4 }
       ]
   }

9. Store in Redis Cache:
   SET "station:99.9:full" = JSON.stringify(response_data)
   EXPIRE 600  // 10 minutes

10. Log Analytics Event:
    INSERT INTO analytics_events (
        session_id,
        event_type: "station_load",
        artist_id,
        timestamp
    )

Response (compressed):
{
    artist: {
        id: "artist_wizkid_001",
        name: "Wizkid",
        station: 99.9,
        bio: "Nigerian superstar...",
        hero_image: "cdn.../wizkid_hero.png",
        deep_etched_image: "cdn.../wizkid_cutout.png"
    },
    scene: {
        id: "scene_lagos_street",
        video_url: "cdn.../lagos_street_med.mp4",
        loop_point: 12.4,
        particles: {
            type: "dust",
            count: 300,
            emission_rate: 20,
            color_start: "#FFA500",
            color_end: "#FF6B00"
        },
        ambient_sound: "cdn.../lagos_ambient.mp3"
    },
    influence_orbs: [
        {
            id: "orb_001",
            artist_id: "artist_fela_004",
            artist_name: "Fela Kuti",
            thumbnail: "cdn.../fela_thumb.png",
            position: { x: 106, y: 75, z: -50 },
            orbit_speed: 0.3,
            size: 60,
            glow_color: "#FFD700"
        },
        {
            id: "orb_002",
            artist_id: "artist_sade_005",
            artist_name: "Sade",
            thumbnail: "cdn.../sade_thumb.png",
            position: { x: -85, y: -45, z: -30 },
            orbit_speed: 0.5,
            size: 50,
            glow_color: "#9370DB"
        }
    ],
    haptic_config: {
        lock_pattern: [...]
    },
    preload_adjacent: [
        { frequency: 98.9, artist_id: "artist_davido_002" },
        { frequency: 100.9, artist_id: "artist_burna_003" }
    ]
}

Time: ~120ms (first load) or ~5ms (cached)
```

---

# PHASE 2: USER SWIPES (THE "WORLD SPIN")

## Step 3: User Performs Horizontal Swipe

### Client Action:
```
User swipes RIGHT with velocity = 850 pixels/second
Client detects gesture
Begins rotating cylinder environment
Applies momentum physics locally
```

### Backend Request #3: Track Swipe Interaction
```
POST /api/session/interaction

Request Body:
{
    session_id: "550e8400-e29b-41d4-a716-446655440000",
    interaction_type: "swipe",
    data: {
        direction: "right",
        velocity: 850,
        from_station: 99.9,
        timestamp: "2025-01-06T14:23:15.342Z"
    }
}

Backend Processing:

1. Validate session exists:
   SELECT * FROM user_sessions 
   WHERE session_id = "550e8400..."
   
   IF not found:
       → Return 401 Unauthorized
   
2. Update session activity:
   UPDATE user_sessions 
   SET last_active = NOW(),
       total_interactions = total_interactions + 1
   WHERE session_id = "550e8400..."

3. Add to interaction history:
   UPDATE user_sessions
   SET constellation_path = constellation_path || jsonb_build_object(
       'type', 'swipe',
       'timestamp', NOW(),
       'from_station', 99.9,
       'velocity', 850
   )
   WHERE session_id = "550e8400..."

4. Predict next station:
   Based on swipe velocity and direction:
   
   velocity_units = 850 / 200  // ~4.25 stations
   predicted_station = 99.9 + (4.25 * 1.0)  // Right = positive
   predicted_station = 104.15
   
   Find nearest actual station:
   SELECT station_frequency 
   FROM artists 
   WHERE station_frequency >= 104.15
   ORDER BY station_frequency ASC
   LIMIT 1
   
   Result: 105.5 (Burna Boy)

5. Preload prediction to Redis:
   LPUSH "preload_queue:550e8400..." "station:105.5:full"
   
6. Background job picks up queue and warms cache

Response:
{
    success: true,
    predicted_destination: 105.5
}

Time: ~20ms (non-blocking, client continues animation)
```

---

## Step 4: Swipe Settles, Station Locks

### Client Action:
```
Momentum slows due to friction physics
Cylinder rotation decelerates
Station 105.5 (Burna Boy) snaps into "lock" position
Haptic feedback fires
Client needs new station data
```

### Backend Request #4: Load New Station
```
GET /api/stations/105.5

Backend Processing:

1. Check Redis Cache:
   GET "station:105.5:full"
   
   Result: CACHE HIT (we preloaded it!)
   
2. Return cached data immediately

3. Background: Update analytics
   INSERT INTO station_visits (
       session_id,
       artist_id: "artist_burna_003",
       visited_at: NOW(),
       came_from: "artist_wizkid_001",
       method: "swipe"
   )

4. Check if this creates a new connection:
   IF user has both Wizkid and Burna in constellation:
       → Create connection line in star map
       → Calculate line strength based on transition count

Response (same structure as Step 2, different data):
{
    artist: {
        id: "artist_burna_003",
        name: "Burna Boy",
        station: 105.5,
        ...
    },
    scene: {
        id: "scene_shrine",
        video_url: "cdn.../shrine_med.mp4",
        particles: {
            type: "smoke",
            count: 500,
            rise_speed: 1.2,
            color_start: "#808080",
            color_end: "#404040"
        },
        ...
    },
    influence_orbs: [
        {
            artist_name: "Fela Kuti",  // Also influenced Burna
            position: { x: 90, y: 60, z: -40 },
            ...
        },
        {
            artist_name: "Damian Marley",
            position: { x: -70, y: -30, z: -35 },
            ...
        }
    ],
    ...
}

Time: ~8ms (from cache)
```

---

# PHASE 3: USER TILTS PHONE (GYROSCOPE PARALLAX)

## Step 5: User Tilts Phone to Explore Depth

### Client Action:
```
CoreMotion detects:
- X rotation: 12 degrees (phone tilted right)
- Y rotation: -8 degrees (phone tilted down)

Client processes locally:
- Applies low-pass filter to smooth jitter
- Calculates parallax offsets for each layer:
  * Background video: moves 5px right, 3px down
  * Smoke particles: move 12px right, 7px down
  * Burna cutout: moves 18px right, 10px down
- Renders new frame at 60fps

No immediate backend request needed (client-side processing)
```

### Backend Request #5: Log Sensor Engagement (Batched)
```
Every 30 seconds, client sends batch update:

POST /api/session/sensor-batch

Request Body:
{
    session_id: "550e8400...",
    sensor_data: [
        {
            timestamp: "2025-01-06T14:23:45.120Z",
            type: "gyroscope",
            duration: 8.5,  // seconds of active tilting
            max_tilt_x: 15,
            max_tilt_y: 12
        },
        {
            timestamp: "2025-01-06T14:24:22.450Z",
            type: "gyroscope",
            duration: 5.2,
            max_tilt_x: 10,
            max_tilt_y: 8
        }
    ]
}

Backend Processing:

1. Validate session

2. Update sensor engagement counter:
   UPDATE user_sessions
   SET sensor_engagement_count = sensor_engagement_count + 2
   WHERE session_id = "550e8400..."

3. Store detailed metrics for analytics:
   INSERT INTO sensor_analytics (
       session_id,
       total_duration: 13.7,  // 8.5 + 5.2
       avg_tilt_x: 12.5,
       avg_tilt_y: 10,
       recorded_at: NOW()
   )

4. Check if user has discovered parallax feature:
   IF sensor_engagement_count >= 3 AND user.seen_parallax_tip = false:
       → Queue notification: "You've discovered depth! Try tilting more..."
       → UPDATE users SET seen_parallax_tip = true

Response:
{
    success: true,
    total_sensor_time: 47.3  // Total across all sessions
}

Time: ~15ms
```

---

# PHASE 4: USER TAPS ORB (THE "ZOOM")

## Step 6: User Taps "Fela Kuti" Orb

### Client Action:
```
User taps at screen coordinates (x: 320, y: 480)
Client performs ray-cast into 3D scene
Detects hit on orb with id: "orb_003" (Fela Kuti)
Begins camera zoom animation
```

### Backend Request #6: Initiate Orb Transition
```
POST /api/transition/zoom

Request Body:
{
    session_id: "550e8400...",
    from_artist_id: "artist_burna_003",
    to_artist_id: "artist_fela_004",
    orb_id: "orb_003",
    interaction_point: { x: 320, y: 480 }
}

Backend Processing:

1. Validate orb relationship exists:
   SELECT * FROM influence_orbs
   WHERE id = "orb_003"
     AND source_artist_id = "artist_burna_003"
     AND target_artist_id = "artist_fela_004"
   
   IF not found:
       → Return 404 "Invalid orb transition"

2. Load destination artist (Fela):
   SELECT * FROM artists WHERE id = "artist_fela_004"
   
   Result:
   {
       name: "Fela Kuti",
       station_frequency: 102.7,
       scene_id: "scene_shrine_original",
       ...
   }

3. Calculate camera transition path:
   start_position = { x: 0, y: 0, z: 0 }  // Current camera
   orb_position = { x: 90, y: 60, z: -40 }  // Where orb is
   end_position = { x: 0, y: 0, z: 0 }  // Reset to center for new artist
   
   Generate bezier curve keyframes:
   keyframes = [
       { time: 0.0, pos: start_position, fov: 60 },
       { time: 0.3, pos: orb_position, fov: 45 },  // Zoom toward orb
       { time: 0.5, pos: orb_position, fov: 30 },  // Close-up on orb
       { time: 0.8, pos: end_position, fov: 60 }   // Pull back to new scene
   ]

4. Load full scene data for Fela:
   (Same process as Step 2, but for Fela's station)
   
   Check cache first:
   GET "station:102.7:full"
   
   IF miss:
       → Query database and build response

5. Generate audio crossfade timing:
   crossfade_config = {
       fade_out_start: 0.2,  // Start fading Burna's audio
       fade_out_end: 0.5,
       fade_in_start: 0.5,   // Start fading Fela's audio
       fade_in_end: 0.8,
       curve: "ease_in_out"
   }

6. Update session path:
   UPDATE user_sessions
   SET constellation_path = constellation_path || jsonb_build_object(
       'type', 'orb_zoom',
       'timestamp', NOW(),
       'from', 'artist_burna_003',
       'to', 'artist_fela_004',
       'relationship', 'ancestor'
   ),
   total_orbs_explored = total_orbs_explored + 1
   WHERE session_id = "550e8400..."

7. Check achievement triggers:
   IF total_orbs_explored = 1:
       → Award: "First Discovery" badge
   IF total_orbs_explored = 10:
       → Award: "Deep Diver" badge

8. Analyze discovery path for recommendations:
   Current path: Wizkid → Burna → Fela
   Pattern: Afrobeats → Afrofusion → Afrobeat (Original)
   
   Suggest next: James Brown, Tony Allen (Fela's drummer)
   Store recommendations in Redis for quick access

Response:
{
    transition: {
        duration: 0.8,  // seconds
        camera_path: [
            { time: 0.0, position: {x:0, y:0, z:0}, fov: 60 },
            { time: 0.3, position: {x:90, y:60, z:-40}, fov: 45 },
            { time: 0.5, position: {x:90, y:60, z:-40}, fov: 30 },
            { time: 0.8, position: {x:0, y:0, z:0}, fov: 60 }
        ],
        audio_crossfade: {
            fade_out_start: 0.2,
            fade_out_end: 0.5,
            fade_in_start: 0.5,
            fade_in_end: 0.8
        }
    },
    destination: {
        artist: {
            id: "artist_fela_004",
            name: "Fela Kuti",
            station: 102.7,
            era: "1970s-1990s",
            bio: "Pioneer of Afrobeat...",
            hero_image: "cdn.../fela_hero.png",
            deep_etched_image: "cdn.../fela_cutout.png"
        },
        scene: {
            id: "scene_shrine_original",
            name: "The Shrine",
            video_url: "cdn.../shrine_70s_med.mp4",
            particles: {
                type: "smoke",
                count: 800,
                rise_speed: 0.8,
                turbulence: 1.5,
                color_start: "#696969",
                color_end: "#2F4F4F"
            },
            lighting: {
                ambient_color: "#8B4513",
                intensity: 0.7,
                spotlight_angle: 45
            }
        },
        influence_orbs: [
            {
                artist_name: "James Brown",
                relationship: "influence",
                position: { x: -110, y: 40, z: -55 },
                ...
            },
            {
                artist_name: "Tony Allen",
                relationship: "collaborator",
                position: { x: 95, y: -50, z: -45 },
                ...
            }
        ]
    },
    badges_earned: ["First Discovery"],
    recommendations: [
        { artist_id: "artist_james_brown", reason: "Influenced Fela's funk style" },
        { artist_id: "artist_tony_allen", reason: "Fela's legendary drummer" }
    ]
}

Time: ~85ms
```

---

# PHASE 5: USER CAPTURES STAR (PINCH GESTURE)

## Step 7: User Pinches Screen to Save Artist

### Client Action:
```
User performs pinch-close gesture while viewing Fela
Client detects gesture (scale < 0.5)
Plays capture animation
Prepares to save to constellation
```

### Backend Request #7: Save to Constellation
```
POST /api/constellation/capture

Request Body:
{
    session_id: "550e8400...",
    user_id: "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    artist_id: "artist_fela_004",
    capture_timestamp: "2025-01-06T14:28:33.120Z"
}

Backend Processing:

1. Check if artist already in constellation:
   SELECT * FROM constellations
   WHERE user_id = "7c9e6679..."
     AND artist_id = "artist_fela_004"
   
   IF exists:
       → Return 200 with message: "Already in your constellation"
       → Skip remaining steps
   
2. Get user's current discovery path:
   SELECT constellation_path 
   FROM user_sessions 
   WHERE session_id = "550e8400..."
   
   Result: [
       { artist: "Wizkid", station: 99.9, time: "14:23:15" },
       { artist: "Burna Boy", station: 105.5, time: "14:25:42" },
       { artist: "Fela Kuti", station: 102.7, time: "14:27:18" }
   ]
   
   Extract path: ["Wizkid", "Burna Boy", "Fela Kuti"]

3. Calculate position in star map:
   
   Get existing constellation:
   SELECT * FROM constellations 
   WHERE user_id = "7c9e6679..."
   
   Result: [
       { artist: "Wizkid", position: {x: 0, y: 0, z: 0} },
       { artist: "Burna Boy", position: {x: 120, y: 80, z: -50} }
   ]
   
   Calculate new position:
   - Start from last visited artist (Burna Boy)
   - Add vector based on relationship:
     * Ancestor relationship = move backward and up
     * Peer relationship = move horizontally
     * Influenced relationship = move forward and down
   
   relationship = "ancestor"
   base_position = Burna_position
   offset = {
       x: -80,   // Move left (into past)
       y: 100,   // Move up (elder/higher influence)
       z: -30    // Move back (depth)
   }
   
   new_position = {
       x: 120 + (-80) = 40,
       y: 80 + 100 = 180,
       z: -50 + (-30) = -80
   }
   
   Check for collision with existing stars:
   FOR each existing_star:
       distance = sqrt(
           (new_x - existing_x)² + 
           (new_y - existing_y)² + 
           (new_z - existing_z)²
       )
       IF distance < 60:  // Too close
           → Apply jitter: new_position += random(-20, 20)

4. Insert into constellation:
   INSERT INTO constellations (
       id,
       user_id,
       artist_id,
       captured_at,
       discovery_path,
       position_in_galaxy
   ) VALUES (
       uuid_generate_v4(),
       "7c9e6679...",
       "artist_fela_004",
       NOW(),
       '["Wizkid", "Burna Boy", "Fela Kuti"]',
       '{"x": 40, "y": 180, "z": -80}'
   )

5. Create connection lines:
   
   Find connections to existing stars:
   - Direct path: Burna Boy → Fela (relationship: ancestor)
   - Genre similarity: Calculate for all existing stars
   
   FOR each existing_star IN constellation:
       similarity_score = calculate_similarity(
           fela_genres,
           existing_star_genres
       )
       
       IF similarity_score > 0.7 OR existing_star IN discovery_path:
           INSERT INTO constellation_connections (
               user_id,
               from_artist_id,
               to_artist_id,
               strength: similarity_score,
               connection_type: "genre" OR "discovery_path"
           )

6. Generate "fly away" animation path:
   
   Animation: Star flies from capture point to constellation position
   
   start_position = { x: 0, y: 0, z: 0 }  // Center of screen
   end_position = { x: 40, y: 180, z: -80 }  // In star map
   
   Generate arc path:
   control_point = {
       x: (start.x + end.x) / 2,
       y: max(start.y, end.y) + 200,  // Arc upward
       z: (start.z + end.z) / 2
   }
   
   curve = quadratic_bezier(start, control_point, end)
   
   animation_keyframes = sample_curve(curve, 20)  // 20 points

7. Update user statistics:
   UPDATE users
   SET total_stars_captured = total_stars_captured + 1,
       last_capture_date = NOW()
   WHERE id = "7c9e6679..."
   
   Get updated count:
   SELECT total_stars_captured FROM users WHERE id = "7c9e6679..."
   Result: 3

8. Check for milestones:
   IF total_stars_captured = 1:
       badge = "First Star"
   ELIF total_stars_captured = 10:
       badge = "Constellation Builder"
   ELIF total_stars_captured = 50:
       badge = "Galaxy Master"
   
   IF badge:
       INSERT INTO user_badges (user_id, badge_type, earned_at)

9. Trigger background job: Update recommendations
   ENQUEUE recommendation_refresh_job {
       user_id: "7c9e6679...",
       new_artist: "artist_fela_004",
       constellation_context: [...existing artists]
   }

Response:
{
    success: true,
    star: {
        id: "constellation_star_042",
        artist_id: "artist_fela_004",
        artist_name: "Fela Kuti",
        position: { x: 40, y: 180, z: -80 },
        capture_animation: {
            duration: 1.2,
            path: [
                { t: 0.0, pos: {x:0, y:0, z:0} },
                { t: 0.2, pos: {x:8, y:40, z:-10} },
                { t: 0.4, pos: {x:16, y:80, z:-20} },
                { t: 0.6, pos: {x:24, y:120, z:-40} },
                { t: 0.8, pos: {x:32, y:150, z:-60} },
                { t: 1.0, pos: {x:40, y:180, z:-80} }
            ],
            scale_curve: [1.0, 1.2, 0.8, 0.6, 0.4, 0.3],
            glow_intensity: [1.0, 1.5, 1.8, 1.5, 1.0, 0.8]
        }
    },
    connections: [
        {
            from: "Burna Boy",
            to: "Fela Kuti",
            strength: 0.95,
            type: "discovery_path"
        },
        {
            from: "Wizkid",
            to: "Fela Kuti",
            strength: 0.72,
            type: "genre_similarity"
        }
    ],
    total_stars: 3,
    badges_earned: [],
    next_milestone: {
        badge: "Constellation Builder",
        stars_needed: 7,
        progress: 0.3
    }
}

Time: ~95ms
```

---

# PHASE 6: USER OPENS CONSTELLATION JOURNAL

## Step 8: User Navigates to Constellation View

### Client Action:
```
User taps "Constellation" tab in navigation
App transitions from station view to star map
Shows loading animation
```

### Backend Request #8: Load Full Constellation
```
GET /api/constellation/7c9e6679-7425-40de-944b-e07fc1f90ae7

Backend Processing:

1. Check Redis cache:
   GET "constellation:7c9e6679:full"
   
   IF cache miss:
       → Continue to query database

2. Load all user's captured stars:
   SELECT 
       c.*,
       a.name as artist_name,
       a.station_frequency,
       a.genre,
       a.era,
       a.hero_image_url
   FROM constellations c
   JOIN artists a ON c.artist_id = a.id
   WHERE c.user_id = "7c9e6679..."
   ORDER BY c.captured_at ASC
   
   Result: [
       {
           artist: "Wizkid",
           position: {x: 0, y: 0, z: 0},
           captured_at: "2025-01-06T14:23:15",
           genre: "Afrobeats",
           era: "2010s"
       },
       {
           artist: "Burna Boy",
           position: {x: 120, y: 80, z: -50},
           captured_at: "2025-01-06T14:25:42",
           genre: "Afrofusion",
           era: "2010s"
       },
       {
           artist: "Fela Kuti",
           position: {x: 40, y: 180, z: -80},
           captured_at: "2025-01-06T14:28:33",
           genre: "Afrobeat",
           era: "1970s"
       }
   ]

3. Load all connections:
   SELECT 
       cc.*,
       a1.name as from_name,
       a2.name as to_name
   FROM constellation_connections cc
   JOIN artists a1 ON cc.from_artist_id = a1.id
   JOIN artists a2 ON cc.to_artist_id = a2.id
   WHERE cc.user_id = "7c9e6679..."
   
   Result: [
       {
           from: "Wizkid",
           to: "Burna Boy",
           strength: 0.88,
           type: "discovery_path"
       },
       {
           from: "Burna Boy",
           to: "Fela Kuti",
           strength: 0.95,
           type: "discovery_path"
       },
       {
           from: "Wizkid",
           to: "Fela Kuti",
           strength: 0.72,
           type: "genre_similarity"
       }
   ]

4. Calculate statistics:
   
   Total stars: 3
   
   Genre distribution:
   genres = ["Afrobeats", "Afrofusion", "Afrobeat"]
   genre_count = {
       "Afrobeats": 1,
       "Afrofusion": 1,
       "Afrobeat": 1
   }
   
   Era distribution:
   eras = ["2010s", "2010s", "1970s"]
   era_count = {
       "2010s": 2,
       "1970s": 1
   }
   
   Discovery patterns:
   - Most common path direction: Modern → Classic (2 transitions)
   - Average time between captures: 2.6 minutes
   - Deepest historical reach: 1970s

5. Generate visual layout metadata:
   
   Calculate bounding box for camera framing:
   min_x = min(all star x positions) = 0
   max_x = max(all star x positions) = 120
   min_y = min(all star y positions) = 0
   max_y = max(all star y positions) = 180
   min_z = min(all star z positions) = -80
   max_z = max(all star z positions) = 0
   
   Center point = {
       x: (min_x + max_x) / 2 = 60,
       y: (min_y + max_y) / 2 = 90,
       z: (min_z + max_z) / 2 = -40
   }
   
   Suggested camera position = {
       x: center.x,
       y: center.y,
       z: center.z + 300  // Pull back to see all stars
   }
   
   Zoom level to fit all stars:
   spread = max(max_x - min_x, max_y - min_y, max_z - min_z)
   zoom = spread * 1.2  // Add 20% padding

6. Identify clusters (for visual grouping):
   
   Use k-means clustering on star positions:
   clusters = kmeans(positions, k=2)
   
   Result:
   Cluster 1: [Wizkid, Burna Boy]  // Modern Afrobeats
   Cluster 2: [Fela Kuti]  // Classic Afrobeat

7. Generate insights:
   
   Based on constellation data:
   - "You're exploring the evolution of Afrobeat"
   - "From the original Shrine to Lagos street corners"
   - "3 stars captured, spanning 5 decades"
   
   Recommendations:
   SELECT a.* FROM artists a
   WHERE a.genre IN ('Afrobeats', 'Afrobeat', 'Afrofusion')
     AND a.id NOT IN (
         SELECT artist_id FROM constellations WHERE user_id = "7c9e6679..."
     )
   ORDER BY a.popularity_score DESC
   LIMIT 3
   
   Result: [Davido, Tiwa Savage, King Sunny Ade]

8. Cache the result:
   SET "constellation:7c9e6679:full" = JSON.stringify(response)
   EXPIRE 300  // 5 minutes (shorter than station cache)

Response:
{
    stars: [
        {
            id: "star_001",
            artist: {
                id: "artist_wizkid_001",
                name: "Wizkid",
                thumbnail: "cdn.../wizkid_thumb.png",
                genre: "Afrobeats",
                era: "2010s",
                station: 99.9
            },
            position: { x: 0, y: 0, z: 0 },
            captured_at: "2025-01-06T14:23:15.000Z",
            glow_color: "#FFD700",
            size: 1.0  // Base size
        },
        {
            id: "star_002",
            artist: {
                id: "artist_burna_003",
                name: "Burna Boy",
                thumbnail: "cdn.../burna_thumb.png",
                genre: "Afrofusion",
                era: "2010s",
                station: 105.5
            },
            position: { x: 120, y: 80, z: -50 },
            captured_at: "2025-01-06T14:25:42.000Z",
            glow_color: "#FF6B00",
            size: 1.0
        },
        {
            id: "star_003",
            artist: {
                id: "artist_fela_004",
                name: "Fela Kuti",
                thumbnail: "cdn.../fela_thumb.png",
                genre: "Afrobeat",
                era: "1970s",
                station: 102.7
            },
            position: { x: 40, y: 180, z: -80 },
            captured_at: "2025-01-06T14:28:33.000Z",
            glow_color: "#8B4513",
            size: 1.2  // Slightly larger (influential artist)
        }
    ],
    connections: [
        {
            id: "connection_001",
            from_star_id: "star_001",
            to_star_id: "star_002",
            from_position: { x: 0, y: 0, z: 0 },
            to_position: { x: 120, y: 80, z: -50 },
            strength: 0.88,
            type: "discovery_path",
            color: "#4169E1",
            thickness: 2.5
        },
        {
            id: "connection_002",
            from_star_id: "star_002",
            to_star_id: "star_003",
            from_position: { x: 120, y: 80, z: -50 },
            to_position: { x: 40, y: 180, z: -80 },
            strength: 0.95,
            type: "discovery_path",
            color: "#4169E1",
            thickness: 3.0
        },
        {
            id: "connection_003",
            from_star_id: "star_001",
            to_star_id: "star_003",
            from_position: { x: 0, y: 0, z: 0 },
            to_position: { x: 40, y: 180, z: -80 },
            strength: 0.72,
            type: "genre_similarity",
            color: "#9370DB",
            thickness: 1.5,
            style: "dashed"
        }
    ],
    camera_setup: {
        position: { x: 60, y: 90, z: 260 },
        look_at: { x: 60, y: 90, z: -40 },
        field_of_view: 45,
        suggested_zoom: 144  // Based on spread
    },
    clusters: [
        {
            name: "Modern Afrobeats",
            stars: ["star_001", "star_002"],
            center: { x: 60, y: 40, z: -25 },
            color: "#FF6B00"
        },
        {
            name: "Classic Afrobeat",
            stars: ["star_003"],
            center: { x: 40, y: 180, z: -80 },
            color: "#8B4513"
        }
    ],
    statistics: {
        total_stars: 3,
        genre_distribution: [
            { genre: "Afrobeats", count: 1, percentage: 33 },
            { genre: "Afrofusion", count: 1, percentage: 33 },
            { genre: "Afrobeat", count: 1, percentage: 33 }
        ],
        era_distribution: [
            { era: "2010s", count: 2, percentage: 67 },
            { era: "1970s", count: 1, percentage: 33 }
        ],
        journey_stats: {
            first_star: "Wizkid",
            newest_star: "Fela Kuti",
            time_span: "15 minutes",
            deepest_history: "1970s",
            avg_capture_interval: "2.6 minutes"
        }
    },
    insights: [
        "You're exploring the evolution of Afrobeat",
        "From the original Shrine to Lagos street corners",
        "3 stars captured, spanning 5 decades"
    ],
    recommendations: [
        {
            artist_id: "artist_davido_002",
            name: "Davido",
            reason: "Another Afrobeats pioneer",
            station: 98.9,
            thumbnail: "cdn.../davido_thumb.png"
        },
        {
            artist_id: "artist_tiwa_savage_006",
            name: "Tiwa Savage",
            reason: "Queen of Afrobeats",
            station: 103.3,
            thumbnail: "cdn.../tiwa_thumb.png"
        },
        {
            artist_id: "artist_ksa_007",
            name: "King Sunny Ade",
            reason: "Juju music legend, contemporary of Fela",
            station: 101.1,
            thumbnail: "cdn.../ksa_thumb.png"
        }
    ]
}

Time: ~180ms (uncached) or ~12ms (cached)
```

---

# PHASE 7: BACKGROUND PROCESSES

## Step 9: Analytics Processing (Runs Every 5 Minutes)

### Background Job: Session Aggregation
```
CRON JOB: */5 * * * *  // Every 5 minutes

Function: aggregate_session_metrics()

Processing:

1. Get all active sessions (last_active within 10 minutes):
   SELECT * FROM user_sessions
   WHERE last_active > NOW() - INTERVAL '10 minutes'
     AND session_start > NOW() - INTERVAL '2 hours'
   
   Result: 47 active sessions

2. FOR EACH session:
   
   a. Calculate session metrics:
      duration = last_active - session_start
      
      interactions_per_minute = total_interactions / (duration in minutes)
      
      orbs_per_interaction = total_orbs_explored / total_interactions
      
      sensor_engagement_rate = sensor_engagement_count / duration
   
   b. Categorize session quality:
      IF orbs_per_interaction > 0.3:
          session_quality = "high_curiosity"
      ELIF orbs_per_interaction > 0.1:
          session_quality = "medium_curiosity"
      ELSE:
          session_quality = "low_curiosity"
      
      IF sensor_engagement_count > 5:
          immersion_level = "high"
      ELIF sensor_engagement_count > 0:
          immersion_level = "medium"
      ELSE:
          immersion_level = "low"
   
   c. Update aggregated metrics:
      INSERT INTO session_metrics (
          session_id,
          duration_seconds,
          interactions_per_minute,
          orbs_per_interaction,
          sensor_engagement_rate,
          session_quality,
          immersion_level,
          processed_at
      )
   
   d. Update user profile:
      UPDATE users
      SET total_session_time = total_session_time + duration,
          avg_orb_exploration = (
              (avg_orb_exploration * session_count + orbs_per_interaction) 
              / (session_count + 1)
          ),
          session_count = session_count + 1
      WHERE id = session.user_id

3. Calculate global metrics:
   
   Most explored artists today:
   SELECT 
       artist_id,
       COUNT(*) as visit_count
   FROM station_visits
   WHERE visited_at > NOW() - INTERVAL '1 day'
   GROUP BY artist_id
   ORDER BY visit_count DESC
   LIMIT 10
   
   Store in Redis:
   SET "trending:daily" = JSON.stringify(results)
   EXPIRE 3600  // 1 hour
   
   Popular discovery paths:
   SELECT 
       came_from,
       artist_id,
       COUNT(*) as path_count
   FROM station_visits
   WHERE visited_at > NOW() - INTERVAL '7 days'
       AND came_from IS NOT NULL
   GROUP BY came_from, artist_id
   ORDER BY path_count DESC
   LIMIT 20
   
   Store results:
   SET "popular_paths:weekly" = JSON.stringify(results)
   
   Bottleneck detection:
   Find stations with high visits but low orb exploration:
   SELECT 
       artist_id,
       COUNT(DISTINCT session_id) as unique_visitors,
       AVG(
           SELECT COUNT(*) FROM interaction_logs 
           WHERE type = 'orb_tap' AND artist_id = sv.artist_id
       ) as avg_orb_taps
   FROM station_visits sv
   WHERE visited_at > NOW() - INTERVAL '1 day'
   GROUP BY artist_id
   HAVING avg_orb_taps < 0.5
   
   Flag for review:
   INSERT INTO content_alerts (
       alert_type: "low_engagement",
       artist_id,
       metric: "orb_exploration",
       value: avg_orb_taps,
       created_at: NOW()
   )

4. Update recommendation weights:
   
   Based on global patterns:
   UPDATE artists
   SET popularity_score = (
       (visit_count_7d * 0.4) +
       (orb_tap_count_7d * 0.3) +
       (capture_count_7d * 0.3)
   )
   WHERE updated_at < NOW() - INTERVAL '1 hour'

5. Clean up stale data:
   
   Delete abandoned sessions (no activity in 24 hours):
   DELETE FROM user_sessions
   WHERE last_active < NOW() - INTERVAL '24 hours'
     AND user_id IS NULL  // Anonymous users only
   
   Archive old analytics events:
   INSERT INTO analytics_events_archive
   SELECT * FROM analytics_events
   WHERE timestamp < NOW() - INTERVAL '30 days'
   
   DELETE FROM analytics_events
   WHERE timestamp < NOW() - INTERVAL '30 days'

Completion: Log metrics
INSERT INTO job_logs (
    job_name: "aggregate_session_metrics",
    processed_count: 47,
    duration_ms: 1823,
    completed_at: NOW()
)
```

---

## Step 10: Asset Optimization Pipeline (Triggered on Upload)

### Background Job: Process New Artist Assets
```
TRIGGER: ON artist.hero_image_uploaded

Input: {
    artist_id: "artist_new_artist_099",
    raw_image_url: "uploads/raw_images/new_artist_original.jpg"
}

Processing:

1. Download original image:
   image_data = download_from_s3(raw_image_url)
   
   Validate:
   - Min resolution: 2000x2000px
   - Max file size: 20MB
   - Format: JPG, PNG, or TIFF
   
   IF validation fails:
       → Send notification to admin
       → RETURN error

2. Background removal (Deep Etch):
   
   Option A: Use ML API (Remove.bg, Cloudinary AI)
   response = api.remove_background(image_data, {
       type: "person",
       format: "png",
       crop: false,
       scale: "original"
   })
   
   deep_etched_image = response.image_data
   
   Option B: Queue for manual editing
   IF confidence_score < 0.85:
       → INSERT INTO manual_review_queue (
           artist_id,
           task_type: "background_removal",
           priority: "high",
           raw_image_url
       )
       → Send notification to design team
       → Wait for manual processing

3. Generate multiple resolutions:
   
   resolutions = [
       { name: "@3x", size: 2048 },
       { name: "@2x", size: 1024 },
       { name: "@1x", size: 512 },
       { name: "thumbnail", size: 256 }
   ]
   
   FOR EACH resolution:
       resized_image = resize_image(
           deep_etched_image,
           target_size: resolution.size,
           maintain_aspect: true,
           interpolation: "lanczos"
       )
       
       Optimize:
       IF format == "png":
           compressed = pngquant(resized_image, {
               quality: "85-95",
               speed: 1
           })
       
       File size check:
       IF compressed_size > target_size:
           → Reduce quality iteratively
           WHILE size > target AND quality > 70:
               quality -= 5
               compressed = recompress(image, quality)
       
       filename = "artist_{id}_{name}.png"
       upload_to_cdn(compressed, filename)
       
       Store URL:
       urls[resolution.name] = cdn_url

4. Generate color palette (for UI theming):
   
   Extract dominant colors:
   palette = extract_color_palette(deep_etched_image, {
       count: 5,
       quality: 10
   })
   
   Result: [
       "#8B4513",  // Primary (from clothing)
       "#FFD700",  // Accent (from jewelry)
       "#2F4F4F",  // Shadow
       "#F5DEB3",  // Highlight
       "#696969"   // Neutral
   ]
   
   Calculate complementary colors for orbs:
   orb_glow_color = calculate_complementary(palette[0])

5. Edge refinement:
   
   Apply anti-aliasing to cutout edges:
   refined_image = apply_edge_smoothing(deep_etched_image, {
       blur_radius: 0.5,
       feather: 1.0
   })
   
   Add subtle glow for depth:
   IF scene_type == "dark":
       add_rim_lighting(refined_image, {
           intensity: 0.3,
           color: "#FFFFFF",
           angle: 135  // Top-left
       })

6. Update database:
   UPDATE artists
   SET hero_image_url = urls["@2x"],
       deep_etched_image_url = urls["@3x"],
       thumbnail_url = urls["thumbnail"],
       color_palette = palette,
       orb_glow_color = orb_glow_color,
       asset_status = "ready",
       assets_processed_at = NOW()
   WHERE id = artist_id

7. Invalidate caches:
   DELETE FROM redis WHERE key LIKE "station:{artist.station}*"
   DELETE FROM redis WHERE key LIKE "artist:{artist_id}*"

8. Notify completion:
   publish_event("artist_assets_ready", {
       artist_id,
       urls,
       processing_time_seconds: elapsed_time
   })
   
   Send webhook to admin dashboard:
   POST https://admin.deepstream.io/webhooks/assets
   {
       artist_name: "New Artist",
       status: "ready",
       preview_url: urls["@1x"]
   }

Completion time: 45-120 seconds per artist
```

---

## Step 11: Influence Graph Builder (Runs Daily)

### Background Job: Update Artist Relationships
```
CRON JOB: 0 3 * * *  // 3 AM daily

Function: build_influence_graph()

Processing:

1. Get all artists needing relationship updates:
   SELECT * FROM artists
   WHERE influence_graph_updated_at < NOW() - INTERVAL '7 days'
      OR influence_graph_updated_at IS NULL
   ORDER BY popularity_score DESC
   LIMIT 50  // Process 50 per day
   
   Result: 50 artists

2. FOR EACH artist:
   
   a. Query external APIs for relationships:
      
      MusicBrainz API:
      response = fetch(
          "https://musicbrainz.org/ws/2/artist/{mbid}/relations",
          { format: "json" }
      )
      
      Extract relationships:
      influences = []
      FOR EACH relation IN response.relations:
          IF relation.type == "influenced by":
              influences.push({
                  name: relation.target.name,
                  confidence: 1.0  // Explicit relationship
              })
      
      Spotify API:
      response = fetch(
          "https://api.spotify.com/v1/artists/{spotify_id}/related-artists"
      )
      
      related_artists = response.artists.map(a => ({
          name: a.name,
          confidence: 0.7  // Algorithmic relationship
      }))
      
      Last.fm API:
      response = fetch(
          "http://ws.audioscrobbler.com/2.0/?method=artist.getsimilar",
          { artist: artist.name, limit: 20 }
      )
      
      similar_artists = response.similarartists.artist.map(a => ({
          name: a.name,
          confidence: parseFloat(a.match)  // 0.0 - 1.0
      }))
   
   b. Combine and score relationships:
      
      all_relationships = [...influences, ...related_artists, ...similar_artists]
      
      Group by artist name:
      grouped = {}
      FOR EACH rel IN all_relationships:
          IF grouped[rel.name]:
              grouped[rel.name].confidence += rel.confidence
              grouped[rel.name].source_count++
          ELSE:
              grouped[rel.name] = {
                  confidence: rel.confidence,
                  source_count: 1
              }
      
      Normalize confidence:
      FOR EACH name, data IN grouped:
          data.confidence = data.confidence / data.source_count
   
   c. Match to existing artists in database:
      
      FOR EACH potential_influence IN grouped:
          Search database:
          SELECT * FROM artists
          WHERE SIMILARITY(name, potential_influence.name) > 0.8
             OR name ILIKE '%' || potential_influence.name || '%'
          
          IF match found:
              matched_influences.push({
                  target_artist_id: match.id,
                  confidence: potential_influence.confidence,
                  source_count: potential_influence.source_count
              })
   
   d. Calculate relationship metadata:
      
      FOR EACH influence IN matched_influences:
          
          Get target artist:
          target = SELECT * FROM artists WHERE id = influence.target_artist_id
          
          Determine relationship type:
          IF target.era < artist.era:
              relationship_type = "ancestor"
          ELIF target.era == artist.era:
              IF abs(target.station - artist.station) < 5:
                  relationship_type = "peer"
              ELSE:
                  relationship_type = "contemporary"
          ELSE:
              relationship_type = "influenced"
          
          Calculate genre overlap:
          artist_genres = artist.genre.split(", ")
          target_genres = target.genre.split(", ")
          overlap = intersection(artist_genres, target_genres).length
          genre_similarity = overlap / max(len(artist_genres), len(target_genres))
          
          Final influence score:
          score = (
              influence.confidence * 0.4 +
              genre_similarity * 0.3 +
              (1 / abs(target.station - artist.station)) * 0.2 +
              (influence.source_count / 3) * 0.1
          )
          
          IF score > 0.5:  // Threshold for creating orb
              → Create relationship
   
   e. Position orbs in 3D space:
      
      Sort influences by relationship_type:
      ancestors = filter(influences, type == "ancestor")
      peers = filter(influences, type == "peer")
      contemporaries = filter(influences, type == "contemporary")
      
      Layout ancestors (behind and above):
      FOR i, influence IN ancestors:
          angle = (i / ancestors.length) * 180 + 90  // Spread across back
          orbit_radius = 150 + (i * 20)  // Deeper for older
          inclination = 30  // Tilted up
          orbit_speed = 0.2 + (i * 0.05)
          
          orb_config = {
              orbit_radius,
              orbit_speed,
              starting_angle: angle,
              inclination
          }
      
      Layout peers (horizontal ring):
      FOR i, influence IN peers:
          angle = (i / peers.length) * 360  // Full circle
          orbit_radius = 120
          inclination = 0  // Level with artist
          orbit_speed = 0.4
          
          orb_config = {...}
      
      Layout contemporaries (front and below):
      FOR i, influence IN contemporaries:
          angle = (i / contemporaries.length) * 120 + 30  // Front arc
          orbit_radius = 100
          inclination = -20  // Tilted down
          orbit_speed = 0.5
          
          orb_config = {...}
   
   f. Create or update orb records:
      
      FOR EACH influence WITH orb_config:
          
          Check if exists:
          existing = SELECT * FROM influence_orbs
          WHERE source_artist_id = artist.id
            AND target_artist_id = influence.target_artist_id
          
          IF existing:
              UPDATE influence_orbs
              SET orbit_config = orb_config,
                  relationship_type = relationship_type,
                  influence_score = score,
                  updated_at = NOW()
              WHERE id = existing.id
          ELSE:
              INSERT INTO influence_orbs (
                  id,
                  source_artist_id,
                  target_artist_id,
                  relationship_type,
                  orbit_config,
                  orb_size: 50 + (score * 30),  // 50-80px
                  orb_color: calculate_orb_color(relationship_type),
                  glow_intensity: score,
                  influence_score: score,
                  connection_description: generate_description(artist, target),
                  created_at: NOW()
              )
   
   g. Generate connection descriptions:
      
      Function: generate_description(source, target)
      
      Templates based on relationship:
      IF type == "ancestor":
          templates = [
              "{target} pioneered the sound that {source} modernized",
              "{source} draws from {target}'s revolutionary approach",
              "The lineage from {target} to {source} is unmistakable"
          ]
      ELIF type == "peer":
          templates = [
              "{source} and {target} rose together in the {era}",
              "Contemporaries pushing {genre} forward",
              "Mutual influence between two {genre} innovators"
          ]
      
      Fill template:
      description = random_choice(templates).format(
          source=source.name,
          target=target.name,
          era=source.era,
          genre=source.genre
      )
      
      RETURN description
   
   h. Update artist record:
      UPDATE artists
      SET influence_graph_updated_at = NOW(),
          total_influences = count(matched_influences)
      WHERE id = artist.id
   
   i. Invalidate caches:
      DELETE FROM redis WHERE key = "station:{artist.station}:full"
      DELETE FROM redis WHERE key = "orbs:{artist.id}*"

3. Log completion:
   INSERT INTO job_logs (
       job_name: "build_influence_graph",
       artists_processed: 50,
       orbs_created: total_orbs_created,
       orbs_updated: total_orbs_updated,
       duration_ms: elapsed_time,
       completed_at: NOW()
   )

Total time: 2-4 hours for 50 artists
```

---

# PHASE 8: ADVANCED FEATURES

## Step 12: Personalized Recommendations Engine

### Background Job: Generate User Recommendations
```
TRIGGER: After user captures new star OR every 24 hours

Input: {
    user_id: "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}

Processing:

1. Load user's constellation:
   constellation = SELECT artist_id FROM constellations
   WHERE user_id = user_id
   
   Result: [
       "artist_wizkid_001",
       "artist_burna_003",
       "artist_fela_004"
   ]

2. Extract user preferences:
   
   Get all artists data:
   user_artists = SELECT * FROM artists 
   WHERE id IN constellation
   
   Genre preferences:
   genres = extract_all(user_artists, "genre")
   genre_frequency = {
       "Afrobeats": 2,
       "Afrobeat": 1,
       "Afrofusion": 1
   }
   
   top_genres = sort_by_frequency(genre_frequency).slice(0, 3)
   Result: ["Afrobeats", "Afrobeat", "Afrofusion"]
   
   Era preferences:
   eras = extract_all(user_artists, "era")
   era_distribution = {
       "2010s": 2,
       "1970s": 1
   }
   
   Geographic preferences:
   origins = extract_all(user_artists, "origin")
   Result: ["Lagos, Nigeria", "Lagos, Nigeria", "Lagos, Nigeria"]
   → Strong preference for Nigerian artists

3. Find unexplored influences:
   
   Get all orbs from user's constellation artists:
   connected_artists = SELECT DISTINCT target_artist_id
   FROM influence_orbs
   WHERE source_artist_id IN constellation
   
   Filter out already captured:
   unexplored = connected_artists - constellation
   
   Result: [
       "artist_davido_002",
       "artist_james_brown_008",
       "artist_tony_allen_009",
       "artist_sade_005"
   ]
   
   Score each:
   FOR EACH artist IN unexplored:
       score = 0
       
       # Genre match
       IF artist.genre IN top_genres:
           score += 30
       
       # Connection strength
       connection_count = COUNT orbs pointing to this artist
       score += connection_count * 10
       
       # Era diversity bonus
       IF artist.era NOT IN user's eras:
           score += 15  // Encourage exploration
       
       # Popularity boost (but not too much)
       score += artist.popularity_score * 0.05
       
       recommendations.push({ artist, score })

4. Find similar artists (collaborative filtering):
   
   Get other users with similar constellations:
   similar_users = SELECT 
       c.user_id,
       COUNT(*) as overlap_count
   FROM constellations c
   WHERE c.artist_id IN (
       SELECT artist_id FROM constellations WHERE user_id = user_id
   )
   AND c.user_id != user_id
   GROUP BY c.user_id
   HAVING COUNT(*) >= 2  // At least 2 artists in common
   ORDER BY overlap_count DESC
   LIMIT 20
   
   Result: [
       { user_id: "user_abc", overlap: 2 },
       { user_id: "user_def", overlap: 2 },
       ...
   ]
   
   Get their unique captures:
   collaborative_recommendations = SELECT 
       c.artist_id,
       COUNT(*) as frequency
   FROM constellations c
   WHERE c.user_id IN (similar_users)
     AND c.artist_id NOT IN (current_user_constellation)
   GROUP BY c.artist_id
   ORDER BY frequency DESC
   LIMIT 10
   
   Result: [
       "artist_tiwa_savage_006",  // 8 similar users have this
       "artist_ksa_007",          // 6 similar users have this
       ...
   ]
   
   Score these:
   FOR EACH artist IN collaborative_recommendations:
       score = frequency * 20  // More popular = higher score
       recommendations.push({ artist, score, reason: "Users like you explored this" })

5. Find trending artists:
   
   trending = GET "trending:daily" FROM redis
   
   FOR EACH trending_artist:
       IF trending_artist NOT IN constellation:
           score = 25
           recommendations.push({ 
               artist, 
               score, 
               reason: "Trending today" 
           })

6. Era-based discovery:
   
   Get artists from unexplored eras:
   all_eras = ["1960s", "1970s", "1980s", "1990s", "2000s", "2010s", "2020s"]
   explored_eras = extract_all(user_artists, "era")
   unexplored_eras = all_eras - explored_eras
   
   FOR EACH era IN unexplored_eras:
       era_artists = SELECT * FROM artists
       WHERE era = era
         AND genre IN top_genres
       ORDER BY popularity_score DESC
       LIMIT 3
       
       FOR EACH artist IN era_artists:
           score = 20
           recommendations.push({
               artist,
               score,
               reason: "Explore {era} {genre}"
           })

7. Combine and rank all recommendations:
   
   all_recommendations = [
       ...unexplored_influences,
       ...collaborative_recommendations,
       ...trending,
       ...era_based
   ]
   
   Remove duplicates (keep highest score):
   unique_recommendations = deduplicate_by_artist_id(all_recommendations)
   
   Sort by score:
   sorted_recommendations = sort(unique_recommendations, by: score, desc: true)
   
   Take top 10:
   final_recommendations = sorted_recommendations.slice(0, 10)

8. Add contextual metadata:
   
   FOR EACH rec IN final_recommendations:
       
       Load artist data:
       artist = SELECT * FROM artists WHERE id = rec.artist_id
       
       Find connection to user's constellation:
       connection = SELECT * FROM influence_orbs
       WHERE target_artist_id = rec.artist_id
         AND source_artist_id IN constellation
       LIMIT 1
       
       IF connection:
           rec.connection_text = "Connected to {source_artist_name}"
       
       Add preview data:
       rec.preview_image = artist.thumbnail_url
       rec.station = artist.station_frequency
       rec.genre = artist.genre

9. Store in cache:
   SET "recommendations:{user_id}" = JSON.stringify(final_recommendations)
   EXPIRE 86400  // 24 hours

10. Return or queue for push notification:
    IF user has notifications enabled:
        top_rec = final_recommendations[0]
        
        ENQUEUE push_notification {
            user_id,
            title: "New artist to explore",
            body: "{top_rec.artist_name} - {top_rec.reason}",
            deep_link: "deepstream://station/{top_rec.station}"
        }

Response (cached or new):
{
    recommendations: [
        {
            artist: {
                id: "artist_davido_002",
                name: "Davido",
                thumbnail: "cdn.../davido_thumb.png",
                station: 98.9,
                genre: "Afrobeats",
                era: "2010s"
            },
            score: 65,
            reason: "Connected to Wizkid",
            connection_text: "Peer and collaborator of Wizkid",
            confidence: 0.85
        },
        {
            artist: {
                id: "artist_james_brown_008",
                name: "James Brown",
                thumbnail: "cdn.../jb_thumb.png",
                station: 96.3,
                genre: "Funk, Soul",
                era: "1960s"
            },
            score: 58,
            reason: "Influenced Fela Kuti",
            connection_text: "The Godfather of Soul inspired Fela's funk",
            confidence: 0.92
        },
        {
            artist: {
                id: "artist_tiwa_savage_006",
                name: "Tiwa Savage",
                thumbnail: "cdn.../tiwa_thumb.png",
                station: 103.3,
                genre: "Afrobeats, R&B",
                era: "2010s"
            },
            score: 55,
            reason: "Users like you explored this",
            connection_text: "Queen of Afrobeats",
            confidence: 0.78
        },
        // ... 7 more
    ],
    generated_at: "2025-01-06T14:35:12.000Z",
    refresh_in: 86400  // seconds
}

Time: ~250ms (first generation) or ~10ms (cached)
```

---

## Step 13: Search Functionality

### User Action: Searches for Artist
```
GET /api/search?q=nina+simone&limit=10

Backend Processing:

1. Parse and sanitize query:
   query = "nina simone"
   query_terms = split(query, " ")
   Result: ["nina", "simone"]
   
   Sanitize:
   query = escape_sql_injection(query)
   query = remove_special_chars(query)

2. Check search cache:
   cache_key = "search:{md5(query)}:10"
   cached_result = GET cache_key FROM redis
   
   IF cached_result:
       → Return immediately

3. Execute full-text search:
   
   PostgreSQL with trigram similarity:
   results = SELECT 
       a.*,
       SIMILARITY(a.name, query) as name_match,
       SIMILARITY(a.biography, query) as bio_match,
       SIMILARITY(a.genre, query) as genre_match,
       (
           SIMILARITY(a.name, query) * 3.0 +
           SIMILARITY(a.biography, query) * 1.0 +
           SIMILARITY(a.genre, query) * 1.5
       ) as relevance_score
   FROM artists a
   WHERE 
       a.name % query  // Trigram operator
       OR a.biography % query
       OR a.genre % query
   ORDER BY relevance_score DESC
   LIMIT limit
   
   Alternative: Elasticsearch query:
   results = elasticsearch.search({
       index: "artists",
       body: {
           query: {
               multi_match: {
                   query: query,
                   fields: ["name^3", "biography", "genre^1.5"],
                   fuzziness: "AUTO"
               }
           }
       },
       size: limit
   })

4. Boost results based on user preferences:
   
   IF user_id provided:
       user_genres = get_user_genre_preferences(user_id)
       
       FOR EACH result IN results:
           IF result.genre IN user_genres:
               result.relevance_score *= 1.2  // 20% boost
           
           IF result.id IN user_constellation:
               result.relevance_score *= 0.5  // De-rank already captured

5. Add contextual data:
   
   FOR EACH result IN results:
       
       Check if in user's constellation:
       result.in_constellation = check_if_captured(user_id, result.id)
       
       Get influence count:
       result.influence_count = COUNT(
           SELECT * FROM influence_orbs WHERE source_artist_id = result.id
       )
       
       Get scene preview:
       result.scene_preview = SELECT video_url_low FROM scenes
       WHERE id = result.scene_id

6. Log search analytics:
   INSERT INTO search_logs (
       user_id,
       session_id,
       query,
       result_count,
       top_result_id,
       timestamp
   )

7. Cache results:
   SET cache_key = JSON.stringify(results)
   EXPIRE 1800  // 30 minutes

Response:
{
    query: "nina simone",
    results: [
        {
            artist: {
                id: "artist_nina_simone_010",
                name: "Nina Simone",
                station: 97.5,
                genre: "Jazz, Soul, Blues",
                era: "1960s",
                thumbnail: "cdn.../nina_thumb.png",
                bio_snippet: "The High Priestess of Soul..."
            },
            relevance_score: 0.95,
            in_constellation: false,
            influence_count: 12,
            scene_preview: "cdn.../jazz_club_low.mp4"
        }
    ],
    total_results: 1,
    search_time_ms: 45
}

Time: ~50ms (uncached) or ~5ms (cached)
```

---

## Step 14: Session Resume (User Returns After Closing App)

### User Action: Reopens App After 2 Hours
```
POST /api/session/resume

Request Body:
{
    device_id: "iPhone_14_Pro_ABC123",
    last_session_id: "550e8400-e29b-41d4-a716-446655440000"
}

Backend Processing:

1. Check if last session is still active:
   last_session = SELECT * FROM user_sessions
   WHERE session_id = last_session_id
   
   IF last_active > NOW() - INTERVAL '2 hours':
       → Session still valid, reactivate it
   ELSE:
       → Session expired, create new one

2. Reactivate or create session:
   
   IF session valid:
       UPDATE user_sessions
       SET last_active = NOW(),
           resume_count = resume_count + 1
       WHERE session_id = last_session_id
       
       session_id = last_session_id
   ELSE:
       INSERT INTO user_sessions (
           session_id,
           user_id,
           device_id,
           session_start,
           last_active
       )
       
       session_id = new_session_id

3. Load last known state:
   
   Get last visited artist:
   last_visit = SELECT artist_id, current_station
   FROM user_sessions
   WHERE session_id = session_id OR session_id = last_session_id
   ORDER BY last_active DESC
   LIMIT 1
   
   Result: {
       artist_id: "artist_fela_004",
       station: 102.7
   }

4. Check for updates since last session:
   
   New recommendations:
   recommendations_count = COUNT(
       SELECT * FROM recommendations
       WHERE user_id = user_id
         AND created_at > last_active
   )
   
   New artists added:
   new_artists = SELECT COUNT(*) FROM artists
   WHERE created_at > last_active
   
   Constellation updates from similar users:
   updates_count = get_social_updates(user_id, last_active)

5. Generate "welcome back" message:
   
   time_away = NOW() - last_active
   
   IF time_away < 1 hour:
       message = "Welcome back! Right where you left off."
   ELIF time_away < 24 hours:
       message = "Welcome back! Ready to continue exploring?"
   ELSE:
       days_away = floor(time_away / 86400)
       message = "Welcome back! You've been away for {days} days. Check out what's new."

6. Prefetch previous station data:
   station_data = GET "station:102.7:full" FROM redis
   
   IF not cached:
       → Trigger background preload

Response:
{
    session_id: "550e8400-e29b-41d4-a716-446655440000",
    is_resumed: true,
    welcome_message: "Welcome back! Right where you left off.",
    last_station: {
        artist_id: "artist_fela_004",
        station: 102.7,
        artist_name: "Fela Kuti"
    },
    updates: {
        new_recommendations: 3,
        new_artists: 0,
        social_updates: 0
    },
    suggested_action: "continue_exploring"  // or "check_recommendations"
}

Time: ~35ms
```

---

## Step 15: Offline Mode Support

### User Action: Loses Internet Connection
```
Scenario: User is on airplane, connection drops

Client Detection:
- iOS Reachability detects network loss
- App enters offline mode
- Shows indicator: "Exploring offline"

Backend Preparation (happens before offline):

1. Predictive caching strategy:
   
   When user loads any station, server includes:
   
   Response header:
   X-Cache-Hints: "preload:98.9,100.9,103.3"
   
   Client automatically prefetches:
   - Adjacent stations (±1-2 frequencies)
   - All orb destination artists
   - Scene videos at medium quality
   - Audio tracks

2. Offline data package structure:
   
   Stored in iOS local storage:
   {
       last_updated: "2025-01-06T14:28:33",
       stations: {
           "99.9": { full artist + scene data },
           "102.7": { full artist + scene data },
           "105.5": { full artist + scene data }
       },
       constellation: {
           stars: [...],
           connections: [...]
       },
       cached_assets: [
           "cdn.../wizkid_hero.png",
           "cdn.../burna_hero.png",
           "cdn.../fela_hero.png",
           "cdn.../lagos_street_med.mp4",
           "cdn.../shrine_med.mp4"
       ]
   }

3. Offline interactions queued:
   
   User actions while offline:
   - Swipe to new station → Check local cache first
   - Tap orb → Load from cache if available
   - Capture star → Queue for sync
   - Gyroscope parallax → Works normally (client-side)
   
   Queued operations stored locally:
   offline_queue = [
       {
           type: "capture_star",
           artist_id: "artist_sade_005",
           timestamp: "2025-01-06T15:12:44",
           session_id: "550e8400..."
       },
       {
           type: "orb_interaction",
           from: "artist_fela_004",
           to: "artist_james_brown_008",
           timestamp: "2025-01-06T15:15:22"
       }
   ]

4. Sync when connection returns:
   
   POST /api/sync/offline-queue
   
   Request Body:
   {
       session_id: "550e8400...",
       offline_duration: 1843,  // seconds
       queued_operations: offline_queue
   }
   
   Backend Processing:
   
   FOR EACH operation IN queue:
       SWITCH operation.type:
           
           CASE "capture_star":
               Check if already captured:
               existing = SELECT * FROM constellations
               WHERE user_id = user_id AND artist_id = operation.artist_id
               
               IF NOT existing:
                   INSERT INTO constellations (...)
                   # Normal capture flow
               ELSE:
                   # Already captured from another device or earlier sync
                   LOG duplicate_capture_attempt
           
           CASE "orb_interaction":
               INSERT INTO interaction_logs (
                   session_id,
                   type: "orb_tap",
                   data: operation,
                   occurred_at: operation.timestamp,
                   synced_at: NOW()
               )
           
           CASE "station_visit":
               INSERT INTO station_visits (
                   session_id,
                   artist_id: operation.artist_id,
                   visited_at: operation.timestamp,
                   method: "offline_cache"
               )
   
   Update session:
   UPDATE user_sessions
   SET total_interactions = total_interactions + queue_length,
       offline_duration = offline_duration,
       last_synced: NOW()
   WHERE session_id = session_id
   
   Response:
   {
       synced_operations: 2,
       conflicts_resolved: 0,
       new_data: {
           recommendations: [...],
           constellation_updates: [...]
       }
   }

Time: ~120ms for 10 queued operations
```

---

## Step 16: Social Features (Shared Constellations)

### User Action: Shares Constellation with Friend
```
POST /api/constellation/share

Request Body:
{
    user_id: "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    share_type: "public_link",  // or "direct_user", "social_media"
    message: "Check out my musical journey!"
}

Backend Processing:

1. Generate shareable link:
   
   share_id = generate_unique_id()  // e.g., "aB3dF9gH"
   
   share_url = "https://deepstream.io/constellation/{share_id}"

2. Create share record:
   INSERT INTO constellation_shares (
       id: share_id,
       user_id,
       constellation_snapshot: (
           SELECT json_agg(c.*) 
           FROM constellations c 
           WHERE c.user_id = user_id
       ),
       share_type,
       message,
       views: 0,
       created_at: NOW(),
       expires_at: NOW() + INTERVAL '30 days'
   )

3. Generate preview metadata (for social media):
   
   constellation = get_constellation(user_id)
   
   stats = {
       total_stars: constellation.stars.length,
       genres: unique(constellation.stars.map(s => s.genre)),
       era_span: max(eras) - min(eras)
   }
   
   Create Open Graph tags:
   og_data = {
       title: "{user_name}'s Musical Constellation",
       description: "{total_stars} artists spanning {era_span} of music",
       image: generate_constellation_preview_image(share_id),
       url: share_url
   }
   
   Background job generates preview image:
   - Render constellation as static image
   - 1200x630px (optimal for social)
   - Upload to CDN
   - Store URL

4. Optional: Send notification to recipient:
   
   IF share_type == "direct_user":
       recipient_user_id = request.body.recipient_id
       
       INSERT INTO notifications (
           user_id: recipient_user_id,
           type: "constellation_share",
           from_user_id: user_id,
           share_id,
           message,
           created_at: NOW()
       )
       
       IF recipient has push notifications enabled:
           ENQUEUE push_notification {
               user_id: recipient_user_id,
               title: "{sender_name} shared their constellation",
               body: message,
               deep_link: share_url
           }

Response:
{
    share_id: "aB3dF9gH",
    share_url: "https://deepstream.io/constellation/aB3dF9gH",
    preview_image: "cdn.../constellation_preview_aB3dF9gH.png",
    expires_at: "2025-02-05T14:35:00.000Z",
    social_metadata: {
        og_title: "My Musical Constellation",
        og_description: "3 artists spanning 5 decades of music",
        og_image: "cdn.../constellation_preview_aB3dF9gH.png"
    }
}

Time: ~85ms (plus background image generation ~3-5 seconds)
```

---

### Viewing Shared Constellation
```
GET /api/constellation/shared/{share_id}

Backend Processing:

1. Validate share ID:
   share = SELECT * FROM constellation_shares
   WHERE id = share_id
     AND (expires_at > NOW() OR expires_at IS NULL)
   
   IF NOT found:
       → Return 404 "Constellation not found or expired"

2. Increment view counter:
   UPDATE constellation_shares
   SET views = views + 1,
       last_viewed_at = NOW()
   WHERE id = share_id

3. Load constellation snapshot:
   constellation_data = share.constellation_snapshot
   
   # Snapshot is frozen at share time, not live

4. Enrich with current artist data:
   
   FOR EACH star IN constellation_data:
       current_artist = SELECT * FROM artists 
       WHERE id = star.artist_id
       
       # Use current images/data in case they've been updated
       star.thumbnail = current_artist.thumbnail_url
       star.station = current_artist.station_frequency

5. Generate comparison data (if viewer is logged in):
   
   IF viewer_user_id:
       viewer_constellation = get_constellation(viewer_user_id)
       
       overlap = intersection(
           constellation_data.stars,
           viewer_constellation.stars
       )
       
       unique_to_shared = constellation_data.stars - overlap
       unique_to_viewer = viewer_constellation.stars - overlap
       
       similarity_score = (
           overlap.length / 
           max(constellation_data.length, viewer_constellation.length)
       )
       
       recommendations = unique_to_shared.slice(0, 5)

Response:
{
    share_id: "aB3dF9gH",
    owner: {
        display_name: "Music Explorer",  // Anonymized unless friends
        avatar_url: "cdn.../avatar_default.png"
    },
    message: "Check out my musical journey!",
    constellation: {
        stars: [...],  // Full constellation data
        connections: [...],
        statistics: {...}
    },
    created_at: "2025-01-06T14:35:00.000Z",
    views: 47,
    viewer_data: {
        is_logged_in: true,
        overlap_count: 1,
        similarity_score: 0.33,
        recommendations: [
            {
                artist: "Burna Boy",
                reason: "In shared constellation but not yours"
            },
            ...
        ]
    }
}

Time: ~95ms
```

---

## Step 17: Admin Dashboard Analytics

### Admin Query: Get Platform-Wide Metrics
```
GET /api/admin/metrics?period=7d

Backend Processing:

1. Authenticate admin:
   token = request.headers["Authorization"]
   admin = verify_admin_token(token)
   
   IF NOT admin:
       → Return 403 Forbidden

2. Aggregate user metrics:
   
   Total users:
   total_users = SELECT COUNT(DISTINCT user_id) 
   FROM user_sessions
   WHERE session_start > NOW() - INTERVAL '7 days'
   
   New users:
   new_users = SELECT COUNT(*) FROM users
   WHERE created_at > NOW() - INTERVAL '7 days'
   
   Daily active users:
   dau = SELECT COUNT(DISTINCT user_id)
   FROM user_sessions
   WHERE last_active > NOW() - INTERVAL '1 day'
   
   Average session duration:
   avg_duration = SELECT AVG(
       EXTRACT(EPOCH FROM (last_active - session_start))
   )
   FROM user_sessions
   WHERE session_start > NOW() - INTERVAL '7 days'

3. Aggregate engagement metrics (PRD KPIs):
   
   Interaction depth:
   avg_orbs_per_session = SELECT AVG(total_orbs_explored)
   FROM user_sessions
   WHERE session_start > NOW() - INTERVAL '7 days'
   
   Distribution:
   orb_distribution = SELECT 
       CASE 
           WHEN total_orbs_explored = 0 THEN '0 orbs'
           WHEN total_orbs_explored BETWEEN 1 AND 2 THEN '1-2 orbs'
           WHEN total_orbs_explored BETWEEN 3 AND 5 THEN '3-5 orbs'
           ELSE '6+ orbs'
       END as bracket,
       COUNT(*) as session_count
   FROM user_sessions
   WHERE session_start > NOW() - INTERVAL '7 days'
   GROUP BY bracket
   
   Sensor engagement:
   sensor_engagement_rate = (
       SELECT COUNT(*) FROM user_sessions
       WHERE sensor_engagement_count > 0
         AND session_start > NOW() - INTERVAL '7 days'
   ) / total_users
   
   Average session length vs benchmark:
   spotify_avg = 25 * 60  // 25 minutes (industry benchmark)
   deepstream_avg = avg_duration
   lift = ((deepstream_avg - spotify_avg) / spotify_avg) * 100

4. Constellation metrics:
   
   Total stars captured:
   total_captures = SELECT COUNT(*) FROM constellations
   WHERE captured_at > NOW() - INTERVAL '7 days'
   
   Captures per user:
   avg_captures = SELECT AVG(capture_count)
   FROM (
       SELECT user_id, COUNT(*) as capture_count
       FROM constellations
       WHERE captured_at > NOW() - INTERVAL '7 days'
       GROUP BY user_id
   ) subquery
   
   Capture rate:
   capture_rate = total_captures / total_sessions

5. Content performance:
   
   Most visited artists:
   top_artists = SELECT 
       a.name,
       a.station_frequency,
       COUNT(*) as visit_count,
       AVG(sv.duration) as avg_time_spent
   FROM station_visits sv
   JOIN artists a ON sv.artist_id = a.id
   WHERE sv.visited_at > NOW() - INTERVAL '7 days'
   GROUP BY a.id
   ORDER BY visit_count DESC
   LIMIT 10
   
   Most explored orbs:
   top_orbs = SELECT 
       a1.name as from_artist,
       a2.name as to_artist,
       COUNT(*) as click_count
   FROM interaction_logs il
   JOIN influence_orbs io ON il.data->>'orb_id' = io.id::text
   JOIN artists a1 ON io.source_artist_id = a1.id
   JOIN artists a2 ON io.target_artist_id = a2.id
   WHERE il.type = 'orb_tap'
     AND il.occurred_at > NOW() - INTERVAL '7 days'
   GROUP BY io.id, a1.name, a2.name
   ORDER BY click_count DESC
   LIMIT 10
   
   Bottleneck stations (high visits, low engagement):
   bottlenecks = SELECT 
       a.name,
       COUNT(DISTINCT sv.session_id) as visitors,
       AVG(COALESCE(
           (SELECT COUNT(*) FROM interaction_logs 
            WHERE session_id = sv.session_id 
              AND data->>'from_artist_id' = a.id::text),
           0
       )) as avg_interactions
   FROM station_visits sv
   JOIN artists a ON sv.artist_id = a.id
   WHERE sv.visited_at > NOW() - INTERVAL '7 days'
   GROUP BY a.id
   HAVING AVG(COALESCE(...)) < 0.5  // Less than 0.5 interactions per visit
   ORDER BY visitors DESC

6. Technical performance:
   
   Average API response times:
   api_performance = SELECT 
       endpoint,
       AVG(response_time_ms) as avg_response,
       MAX(response_time_ms) as max_response,
       PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms) as p95
   FROM api_logs
   WHERE timestamp > NOW() - INTERVAL '7 days'
   GROUP BY endpoint
   ORDER BY avg_response DESC
   
   Error rates:
   error_rate = SELECT 
       endpoint,
       COUNT(CASE WHEN status_code >= 500 THEN 1 END)::float / COUNT(*) as error_rate
   FROM api_logs
   WHERE timestamp > NOW() - INTERVAL '7 days'
   GROUP BY endpoint
   HAVING error_rate > 0.01  // More than 1% errors

7. Generate time-series data:
   
   Daily breakdown:
   daily_metrics = SELECT 
       DATE(session_start) as date,
       COUNT(DISTINCT user_id) as dau,
       COUNT(*) as sessions,
       AVG(total_orbs_explored) as avg_orbs,
       SUM(CASE WHEN sensor_engagement_count > 0 THEN 1 ELSE 0 END)::float / COUNT(*) as sensor_rate
   FROM user_sessions
   WHERE session_start > NOW() - INTERVAL '7 days'
   GROUP BY DATE(session_start)
   ORDER BY date ASC

Response:{
    period: "7 days",
    generated_at: "2025-01-06T15:00:00.000Z",
    
    user_metrics: {
        total_users: 15234,
        new_users: 1823,
        daily_active_users: 4521,
        avg_session_duration_minutes: 28.4,
        session_duration_lift_vs_benchmark: "+13.6%"
    },
    
    engagement_kpis: {
        interaction_depth: {
            avg_orbs_per_session: 3.2,
            goal: 3.0,
            status: "Meeting goal",
            distribution: {
                "0 orbs": 2341,
                "1-2 orbs": 4123,
                "3-5 orbs": 5234,
                "6+ orbs": 3536
            }
        },
        sensor_engagement: {
            percentage: 67.8,
            goal: 60.0,
            status: "Exceeding goal"
        },
        constellation_growth: {
            capture_rate: 0.42,  // 42% of sessions result in capture
            avg_captures_per_user: 2.8,
            total_captures_7d: 6398
        }
    },
    
    content_performance: {
        top_artists: [
            {
                name: "Burna Boy",
                station: 105.5,
                visits: 2341,
                avg_time_spent: "3:42"
            },
            {
                name: "Wizkid",
                station: 99.9,
                visits: 2198,
                avg_time_spent: "3:18"
            },
            // ... 8 more
        ],
        top_orbs: [
            {
                from: "Burna Boy",
                to: "Fela Kuti",
                clicks: 1823
            },
            // ... 9 more
        ],
    }
}