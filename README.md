# sport_federation_of_rwanda

-- ============================================================
-- RWANDA NATIONAL SPORTS FEDERATION
-- Comprehensive Database Schema with NIDA DOB Verification
-- ============================================================

-- ============================================================
-- SECTION 1: GEOGRAPHIC & ADMINISTRATIVE STRUCTURE
-- ============================================================

CREATE TABLE provinces (
    province_id     SERIAL PRIMARY KEY,
    province_name   VARCHAR(100) NOT NULL UNIQUE,
    province_code   CHAR(3) NOT NULL UNIQUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE districts (
    district_id     SERIAL PRIMARY KEY,
    province_id     INT NOT NULL REFERENCES provinces(province_id),
    district_name   VARCHAR(100) NOT NULL,
    district_code   CHAR(5) NOT NULL UNIQUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sectors (
    sector_id       SERIAL PRIMARY KEY,
    district_id     INT NOT NULL REFERENCES districts(district_id),
    sector_name     VARCHAR(100) NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 2: SPORTS DISCIPLINES
-- ============================================================

CREATE TABLE sport_categories (
    category_id     SERIAL PRIMARY KEY,
    category_name   VARCHAR(100) NOT NULL UNIQUE,  -- e.g. Team Sports, Individual, Combat
    description     TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sports (
    sport_id        SERIAL PRIMARY KEY,
    category_id     INT NOT NULL REFERENCES sport_categories(category_id),
    sport_name      VARCHAR(100) NOT NULL UNIQUE,  -- Football, Basketball, Athletics, etc.
    sport_code      CHAR(5) NOT NULL UNIQUE,
    governing_body  VARCHAR(200),                  -- FIFA, FIBA, World Athletics, etc.
    is_olympic      BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    description     TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE age_categories (
    age_cat_id      SERIAL PRIMARY KEY,
    sport_id        INT NOT NULL REFERENCES sports(sport_id),
    category_name   VARCHAR(50) NOT NULL,          -- U13, U15, U17, U20, Senior, Masters
    min_age         INT NOT NULL,
    max_age         INT,                           -- NULL = no upper limit (Senior/Masters)
    gender          CHAR(1) CHECK (gender IN ('M','F','X')),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(sport_id, category_name, gender)
);

-- ============================================================
-- SECTION 3: FEDERATION HIERARCHY
-- ============================================================

CREATE TABLE federations (
    federation_id   SERIAL PRIMARY KEY,
    sport_id        INT NOT NULL REFERENCES sports(sport_id),
    federation_name VARCHAR(200) NOT NULL,
    acronym         VARCHAR(20),
    federation_type VARCHAR(20) CHECK (federation_type IN ('NATIONAL','PROVINCIAL','DISTRICT')),
    province_id     INT REFERENCES provinces(province_id),
    district_id     INT REFERENCES districts(district_id),
    parent_fed_id   INT REFERENCES federations(federation_id),
    registration_no VARCHAR(100) UNIQUE,
    founding_date   DATE,
    affiliation_date DATE,
    is_active       BOOLEAN DEFAULT TRUE,
    website         VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    address         TEXT,
    logo_url        VARCHAR(500),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 4: CLUBS & TEAMS
-- ============================================================

CREATE TABLE clubs (
    club_id         SERIAL PRIMARY KEY,
    federation_id   INT NOT NULL REFERENCES federations(federation_id),
    sport_id        INT NOT NULL REFERENCES sports(sport_id),
    district_id     INT REFERENCES districts(district_id),
    sector_id       INT REFERENCES sectors(sector_id),
    club_name       VARCHAR(200) NOT NULL,
    club_code       VARCHAR(20) UNIQUE,
    founding_date   DATE,
    registration_no VARCHAR(100) UNIQUE,
    affiliation_status VARCHAR(20) DEFAULT 'ACTIVE'
                    CHECK (affiliation_status IN ('ACTIVE','SUSPENDED','REVOKED','PENDING')),
    home_venue      VARCHAR(255),
    website         VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    address         TEXT,
    logo_url        VARCHAR(500),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE teams (
    team_id         SERIAL PRIMARY KEY,
    club_id         INT NOT NULL REFERENCES clubs(club_id),
    age_cat_id      INT NOT NULL REFERENCES age_categories(age_cat_id),
    team_name       VARCHAR(200) NOT NULL,
    season_year     INT NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(club_id, age_cat_id, season_year)
);

-- ============================================================
-- SECTION 5: NIDA INTEGRATION & IDENTITY VERIFICATION
-- ============================================================

-- Mirror of NIDA data pulled via API (read-only, refreshed periodically)
CREATE TABLE nida_records (
    nida_record_id  SERIAL PRIMARY KEY,
    national_id     CHAR(16) NOT NULL UNIQUE,      -- Rwanda NID: 16 digits
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          CHAR(1) CHECK (gender IN ('M','F')),
    birth_place     VARCHAR(200),
    nationality     VARCHAR(100) DEFAULT 'RWANDAN',
    id_issue_date   DATE,
    id_expiry_date  DATE,
    is_valid        BOOLEAN DEFAULT TRUE,
    last_verified   TIMESTAMP,                     -- Last time we queried NIDA
    raw_response    JSONB,                         -- Store full NIDA API response
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Audit log for every NIDA verification request
CREATE TABLE nida_verification_log (
    log_id          SERIAL PRIMARY KEY,
    national_id     CHAR(16) NOT NULL,
    verification_type VARCHAR(30) NOT NULL         -- REGISTRATION, COMPETITION, PERIODIC_AUDIT
                    CHECK (verification_type IN ('REGISTRATION','COMPETITION_ELIGIBILITY',
                           'PERIODIC_AUDIT','MANUAL_CHECK','TRANSFER')),
    requested_by    INT,                           -- FK to staff/users table
    request_time    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    response_status VARCHAR(20)                    -- SUCCESS, FAILED, MISMATCH, NOT_FOUND
                    CHECK (response_status IN ('SUCCESS','FAILED','MISMATCH','NOT_FOUND','TIMEOUT')),
    dob_submitted   DATE,                          -- DOB submitted by federation/club
    dob_from_nida   DATE,                          -- DOB returned by NIDA
    dob_match       BOOLEAN,                       -- TRUE if they match
    mismatch_days   INT,                           -- How many days difference if mismatch
    notes           TEXT,
    ip_address      INET
);

-- ============================================================
-- SECTION 6: PERSONS (Players, Coaches, Officials, Staff)
-- ============================================================

CREATE TABLE persons (
    person_id       SERIAL PRIMARY KEY,
    national_id     CHAR(16) UNIQUE,               -- Rwanda NID
    nida_record_id  INT REFERENCES nida_records(nida_record_id),

    -- Personal details (sourced from NIDA where possible)
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          CHAR(1) CHECK (gender IN ('M','F','X')),
    nationality     VARCHAR(100) DEFAULT 'RWANDAN',
    birth_place     VARCHAR(200),

    -- Contact
    phone           VARCHAR(30),
    email           VARCHAR(255),
    address         TEXT,
    province_id     INT REFERENCES provinces(province_id),
    district_id     INT REFERENCES districts(district_id),

    -- Media
    photo_url       VARCHAR(500),
    passport_no     VARCHAR(50),
    passport_expiry DATE,

    -- NIDA Verification Status
    nida_verified           BOOLEAN DEFAULT FALSE,
    nida_verification_date  TIMESTAMP,
    nida_dob_match          BOOLEAN,               -- Critical: does DOB match NIDA?
    nida_mismatch_days      INT,                   -- Days difference if mismatch
    nida_status             VARCHAR(20) DEFAULT 'PENDING'
                            CHECK (nida_status IN ('PENDING','VERIFIED','MISMATCH',
                                   'NOT_FOUND','FLAGGED','EXEMPT')),
    nida_flag_reason        TEXT,                  -- Reason if flagged

    -- System
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 7: PLAYERS
-- ============================================================

CREATE TABLE players (
    player_id           SERIAL PRIMARY KEY,
    person_id           INT NOT NULL UNIQUE REFERENCES persons(person_id),
    registration_no     VARCHAR(50) UNIQUE,        -- Federation-issued player ID
    primary_sport_id    INT REFERENCES sports(sport_id),
    player_position     VARCHAR(100),              -- Goalkeeper, Forward, etc.
    dominant_foot       CHAR(1) CHECK (dominant_foot IN ('L','R','B')),  -- Left/Right/Both
    height_cm           DECIMAL(5,1),
    weight_kg           DECIMAL(5,1),
    shirt_number        INT,

    -- Registration
    registration_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    registration_status VARCHAR(20) DEFAULT 'ACTIVE'
                        CHECK (registration_status IN ('ACTIVE','SUSPENDED',
                               'BANNED','RETIRED','PENDING_VERIFICATION')),
    status_reason       TEXT,
    eligible_to_play    BOOLEAN DEFAULT FALSE,     -- Only TRUE after NIDA verification passes

    -- International
    is_international    BOOLEAN DEFAULT FALSE,
    fifa_id             VARCHAR(50),               -- For football players
    international_caps  INT DEFAULT 0,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Player registrations per club per season
CREATE TABLE player_registrations (
    reg_id          SERIAL PRIMARY KEY,
    player_id       INT NOT NULL REFERENCES players(player_id),
    club_id         INT NOT NULL REFERENCES clubs(club_id),
    team_id         INT REFERENCES teams(team_id),
    age_cat_id      INT REFERENCES age_categories(age_cat_id),
    season_year     INT NOT NULL,
    registration_date DATE NOT NULL DEFAULT CURRENT_DATE,
    contract_start  DATE,
    contract_end    DATE,
    reg_status      VARCHAR(20) DEFAULT 'ACTIVE'
                    CHECK (reg_status IN ('ACTIVE','LOANED','TRANSFERRED','EXPIRED','CANCELLED')),
    jersey_number   INT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(player_id, club_id, season_year)
);

-- Player transfers between clubs
CREATE TABLE player_transfers (
    transfer_id     SERIAL PRIMARY KEY,
    player_id       INT NOT NULL REFERENCES players(player_id),
    from_club_id    INT REFERENCES clubs(club_id),
    to_club_id      INT NOT NULL REFERENCES clubs(club_id),
    transfer_date   DATE NOT NULL,
    transfer_type   VARCHAR(20) CHECK (transfer_type IN ('PERMANENT','LOAN','FREE','RETURN_FROM_LOAN')),
    transfer_fee    DECIMAL(15,2),
    currency        CHAR(3) DEFAULT 'RWF',
    approved_by     INT,                           -- FK to staff
    approval_date   DATE,
    status          VARCHAR(20) DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','APPROVED','REJECTED','CANCELLED')),
    notes           TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 8: COACHES & OFFICIALS
-- ============================================================

CREATE TABLE coach_license_types (
    license_type_id SERIAL PRIMARY KEY,
    sport_id        INT REFERENCES sports(sport_id),
    license_code    VARCHAR(20) NOT NULL,          -- CAF C, CAF B, CAF A, PRO, etc.
    license_name    VARCHAR(100) NOT NULL,
    issuing_body    VARCHAR(200),
    min_age         INT DEFAULT 18,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE coaches (
    coach_id        SERIAL PRIMARY KEY,
    person_id       INT NOT NULL UNIQUE REFERENCES persons(person_id),
    license_type_id INT REFERENCES coach_license_types(license_type_id),
    license_no      VARCHAR(100),
    license_issue_date DATE,
    license_expiry_date DATE,
    specialization  VARCHAR(200),
    coaching_status VARCHAR(20) DEFAULT 'ACTIVE'
                    CHECK (coaching_status IN ('ACTIVE','SUSPENDED','BANNED','RETIRED')),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE club_coaches (
    club_coach_id   SERIAL PRIMARY KEY,
    coach_id        INT NOT NULL REFERENCES coaches(coach_id),
    club_id         INT NOT NULL REFERENCES clubs(club_id),
    role            VARCHAR(100),                  -- Head Coach, Assistant, Goalkeeper Coach
    team_id         INT REFERENCES teams(team_id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_current      BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE referees (
    referee_id      SERIAL PRIMARY KEY,
    person_id       INT NOT NULL UNIQUE REFERENCES persons(person_id),
    sport_id        INT REFERENCES sports(sport_id),
    referee_grade   VARCHAR(50),                   -- International, National, Regional, Local
    license_no      VARCHAR(100),
    license_expiry  DATE,
    status          VARCHAR(20) DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE','SUSPENDED','RETIRED','BANNED')),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 9: COMPETITIONS & TOURNAMENTS
-- ============================================================

CREATE TABLE competition_types (
    comp_type_id    SERIAL PRIMARY KEY,
    type_name       VARCHAR(100) NOT NULL,         -- League, Cup, Friendly, Championship
    format          VARCHAR(100)                   -- Round Robin, Knockout, Mixed
);

CREATE TABLE competitions (
    competition_id  SERIAL PRIMARY KEY,
    federation_id   INT NOT NULL REFERENCES federations(federation_id),
    sport_id        INT NOT NULL REFERENCES sports(sport_id),
    age_cat_id      INT REFERENCES age_categories(age_cat_id),
    comp_type_id    INT REFERENCES competition_types(comp_type_id),
    competition_name VARCHAR(200) NOT NULL,
    season_year     INT NOT NULL,
    start_date      DATE,
    end_date        DATE,
    competition_level VARCHAR(30)
                    CHECK (competition_level IN ('NATIONAL','PROVINCIAL','DISTRICT','INTERNATIONAL')),
    status          VARCHAR(20) DEFAULT 'PLANNED'
                    CHECK (status IN ('PLANNED','ONGOING','COMPLETED','CANCELLED','SUSPENDED')),
    max_teams       INT,
    prize_pool      DECIMAL(15,2),
    currency        CHAR(3) DEFAULT 'RWF',
    host_venue      VARCHAR(255),
    description     TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE competition_participants (
    part_id         SERIAL PRIMARY KEY,
    competition_id  INT NOT NULL REFERENCES competitions(competition_id),
    club_id         INT NOT NULL REFERENCES clubs(club_id),
    team_id         INT REFERENCES teams(team_id),
    registration_date DATE DEFAULT CURRENT_DATE,
    entry_fee_paid  BOOLEAN DEFAULT FALSE,
    status          VARCHAR(20) DEFAULT 'REGISTERED'
                    CHECK (status IN ('REGISTERED','CONFIRMED','WITHDRAWN','DISQUALIFIED')),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(competition_id, club_id)
);

CREATE TABLE venues (
    venue_id        SERIAL PRIMARY KEY,
    venue_name      VARCHAR(200) NOT NULL,
    district_id     INT REFERENCES districts(district_id),
    address         TEXT,
    capacity        INT,
    surface_type    VARCHAR(50),                   -- Natural Grass, Artificial Turf, Indoor, etc.
    has_lights      BOOLEAN DEFAULT FALSE,
    latitude        DECIMAL(10,7),
    longitude       DECIMAL(10,7),
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE matches (
    match_id        SERIAL PRIMARY KEY,
    competition_id  INT NOT NULL REFERENCES competitions(competition_id),
    venue_id        INT REFERENCES venues(venue_id),
    home_club_id    INT REFERENCES clubs(club_id),
    away_club_id    INT REFERENCES clubs(club_id),
    match_date      TIMESTAMP,
    round_name      VARCHAR(100),                  -- Group Stage, Quarter Final, Final
    match_number    INT,
    home_score      INT,
    away_score      INT,
    status          VARCHAR(20) DEFAULT 'SCHEDULED'
                    CHECK (status IN ('SCHEDULED','ONGOING','COMPLETED',
                           'POSTPONED','CANCELLED','ABANDONED','FORFEITED')),
    referee_id      INT REFERENCES referees(referee_id),
    attendance      INT,
    match_report    TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 10: PLAYER STATISTICS
-- ============================================================

CREATE TABLE player_match_stats (
    stat_id         SERIAL PRIMARY KEY,
    match_id        INT NOT NULL REFERENCES matches(match_id),
    player_id       INT NOT NULL REFERENCES players(player_id),
    club_id         INT NOT NULL REFERENCES clubs(club_id),
    played          BOOLEAN DEFAULT FALSE,
    minutes_played  INT DEFAULT 0,
    goals           INT DEFAULT 0,
    assists         INT DEFAULT 0,
    yellow_cards    INT DEFAULT 0,
    red_cards       INT DEFAULT 0,
    -- Sport-specific fields stored as JSONB for flexibility
    extra_stats     JSONB,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(match_id, player_id)
);

CREATE TABLE player_season_stats (
    season_stat_id  SERIAL PRIMARY KEY,
    player_id       INT NOT NULL REFERENCES players(player_id),
    club_id         INT NOT NULL REFERENCES clubs(club_id),
    competition_id  INT REFERENCES competitions(competition_id),
    season_year     INT NOT NULL,
    matches_played  INT DEFAULT 0,
    total_minutes   INT DEFAULT 0,
    goals           INT DEFAULT 0,
    assists         INT DEFAULT 0,
    yellow_cards    INT DEFAULT 0,
    red_cards       INT DEFAULT 0,
    extra_stats     JSONB,
    last_updated    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(player_id, club_id, season_year, competition_id)
);

-- ============================================================
-- SECTION 11: DISCIPLINARY MANAGEMENT
-- ============================================================

CREATE TABLE disciplinary_cases (
    case_id         SERIAL PRIMARY KEY,
    person_id       INT NOT NULL REFERENCES persons(person_id),
    club_id         INT REFERENCES clubs(club_id),
    match_id        INT REFERENCES matches(match_id),
    competition_id  INT REFERENCES competitions(competition_id),
    case_type       VARCHAR(50) CHECK (case_type IN (
                        'YELLOW_CARD','RED_CARD','SUSPENSION','BAN',
                        'FINE','AGE_FRAUD','DOPING','MISCONDUCT','APPEAL')),
    incident_date   DATE NOT NULL,
    description     TEXT NOT NULL,
    decision        TEXT,
    suspension_games INT DEFAULT 0,
    suspension_days  INT DEFAULT 0,
    fine_amount     DECIMAL(15,2),
    status          VARCHAR(20) DEFAULT 'OPEN'
                    CHECK (status IN ('OPEN','UNDER_REVIEW','DECIDED','APPEALED','CLOSED')),
    decided_by      INT,
    decision_date   DATE,
    appeal_deadline DATE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 12: AGE FRAUD FLAGS (Core Anti-Fraud Module)
-- ============================================================

CREATE TABLE age_fraud_investigations (
    investigation_id    SERIAL PRIMARY KEY,
    person_id           INT NOT NULL REFERENCES persons(person_id),
    reported_by         INT,                       -- FK to staff/user
    report_date         DATE DEFAULT CURRENT_DATE,
    dob_on_file         DATE NOT NULL,
    dob_from_nida       DATE,
    dob_difference_days INT,                       -- Computed: |dob_on_file - dob_from_nida|
    
    -- Evidence
    evidence_type       VARCHAR(50),               -- NIDA_MISMATCH, PHYSICAL_ASSESSMENT, TIPOFF
    evidence_notes      TEXT,
    
    -- Investigation
    investigation_status VARCHAR(20) DEFAULT 'OPEN'
                        CHECK (investigation_status IN (
                            'OPEN','UNDER_INVESTIGATION','CONFIRMED_FRAUD',
                            'CLEARED','CLOSED','REFERRED_TO_AUTHORITY')),
    outcome             TEXT,
    action_taken        VARCHAR(100),              -- BANNED, SUSPENDED, CLEARED, etc.
    
    -- Linkage
    disciplinary_case_id INT REFERENCES disciplinary_cases(case_id),
    
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 13: SYSTEM USERS & ACCESS CONTROL
-- ============================================================

CREATE TABLE roles (
    role_id         SERIAL PRIMARY KEY,
    role_name       VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE system_users (
    user_id         SERIAL PRIMARY KEY,
    person_id       INT REFERENCES persons(person_id),
    username        VARCHAR(100) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    role_id         INT NOT NULL REFERENCES roles(role_id),
    federation_id   INT REFERENCES federations(federation_id),
    club_id         INT REFERENCES clubs(club_id),
    is_active       BOOLEAN DEFAULT TRUE,
    last_login      TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE audit_log (
    audit_id        SERIAL PRIMARY KEY,
    user_id         INT REFERENCES system_users(user_id),
    action          VARCHAR(100) NOT NULL,
    table_name      VARCHAR(100),
    record_id       INT,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================
-- SECTION 14: INDEXES FOR PERFORMANCE
-- ============================================================

CREATE INDEX idx_persons_national_id     ON persons(national_id);
CREATE INDEX idx_persons_nida_status     ON persons(nida_status);
CREATE INDEX idx_persons_dob             ON persons(date_of_birth);
CREATE INDEX idx_players_status          ON players(registration_status);
CREATE INDEX idx_players_eligible        ON players(eligible_to_play);
CREATE INDEX idx_player_reg_season       ON player_registrations(season_year, club_id);
CREATE INDEX idx_matches_date            ON matches(match_date);
CREATE INDEX idx_matches_competition     ON matches(competition_id);
CREATE INDEX idx_nida_log_nid            ON nida_verification_log(national_id);
CREATE INDEX idx_nida_log_mismatch       ON nida_verification_log(dob_match) WHERE dob_match = FALSE;
CREATE INDEX idx_fraud_status            ON age_fraud_investigations(investigation_status);
CREATE INDEX idx_disciplinary_person     ON disciplinary_cases(person_id);
CREATE INDEX idx_audit_table_record      ON audit_log(table_name, record_id);

-- ============================================================
-- SECTION 15: USEFUL VIEWS
-- ============================================================

-- View: All flagged players with DOB mismatch
CREATE VIEW v_dob_mismatch_players AS
SELECT
    p.person_id,
    p.national_id,
    p.first_name || ' ' || p.last_name      AS full_name,
    p.date_of_birth                          AS registered_dob,
    nr.date_of_birth                         AS nida_dob,
    p.nida_mismatch_days                     AS days_difference,
    p.nida_status,
    pl.registration_status,
    pl.eligible_to_play
FROM persons p
JOIN players pl ON pl.person_id = p.person_id
LEFT JOIN nida_records nr ON nr.nida_record_id = p.nida_record_id
WHERE p.nida_status IN ('MISMATCH','FLAGGED')
ORDER BY ABS(p.nida_mismatch_days) DESC;

-- View: Player registration with club and current eligibility
CREATE VIEW v_player_club_eligibility AS
SELECT
    pl.player_id,
    pl.registration_no,
    p.national_id,
    p.first_name || ' ' || p.last_name      AS full_name,
    p.date_of_birth,
    EXTRACT(YEAR FROM AGE(p.date_of_birth))  AS current_age,
    c.club_name,
    pr.season_year,
    ac.category_name                         AS age_category,
    p.nida_status,
    pl.eligible_to_play,
    pl.registration_status
FROM players pl
JOIN persons p ON p.person_id = pl.person_id
JOIN player_registrations pr ON pr.player_id = pl.player_id
JOIN clubs c ON c.club_id = pr.club_id
LEFT JOIN age_categories ac ON ac.age_cat_id = pr.age_cat_id;

-- View: Competition eligibility check
CREATE VIEW v_competition_eligibility AS
SELECT
    comp.competition_name,
    comp.season_year,
    ac.category_name,
    ac.min_age,
    ac.max_age,
    p.first_name || ' ' || p.last_name      AS player_name,
    p.date_of_birth,
    EXTRACT(YEAR FROM AGE(p.date_of_birth))  AS player_age,
    p.nida_status,
    pl.eligible_to_play,
    CASE
        WHEN p.nida_status != 'VERIFIED' THEN 'BLOCKED - NIDA not verified'
        WHEN EXTRACT(YEAR FROM AGE(p.date_of_birth)) < ac.min_age THEN 'BLOCKED - Too young'
        WHEN ac.max_age IS NOT NULL AND
             EXTRACT(YEAR FROM AGE(p.date_of_birth)) > ac.max_age THEN 'BLOCKED - Too old'
        ELSE 'ELIGIBLE'
    END AS eligibility_status
FROM players pl
JOIN persons p ON p.person_id = pl.person_id
CROSS JOIN competitions comp
JOIN age_categories ac ON ac.age_cat_id = comp.age_cat_id;

-- View: NIDA verification summary dashboard
CREATE VIEW v_nida_verification_summary AS
SELECT
    COUNT(*)                                        AS total_persons,
    COUNT(*) FILTER (WHERE nida_status = 'VERIFIED')    AS verified,
    COUNT(*) FILTER (WHERE nida_status = 'PENDING')     AS pending,
    COUNT(*) FILTER (WHERE nida_status = 'MISMATCH')    AS dob_mismatches,
    COUNT(*) FILTER (WHERE nida_status = 'FLAGGED')     AS flagged,
    COUNT(*) FILTER (WHERE nida_status = 'NOT_FOUND')   AS not_found
FROM persons;

-- ============================================================
-- SECTION 16: SEED DATA - ROLES & SPORT CATEGORIES
-- ============================================================

INSERT INTO roles (role_name, description) VALUES
    ('SUPER_ADMIN',     'Full system access'),
    ('FED_ADMIN',       'National federation administrator'),
    ('PROV_ADMIN',      'Provincial federation administrator'),
    ('CLUB_ADMIN',      'Club administrator'),
    ('REGISTRAR',       'Player registration officer'),
    ('NIDA_OFFICER',    'NIDA verification officer'),
    ('REFEREE_ADMIN',   'Referee and officials manager'),
    ('COMPETITION_MGR', 'Competition and fixtures manager'),
    ('VIEWER',          'Read-only access');

INSERT INTO sport_categories (category_name, description) VALUES
    ('Team Sports',       'Sports played by teams'),
    ('Individual Sports', 'Sports competed individually'),
    ('Combat Sports',     'Martial arts and combat disciplines'),
    ('Aquatic Sports',    'Swimming and water-based sports'),
    ('Racket Sports',     'Tennis, badminton, table tennis'),
    ('Athletics',         'Track and field events');

INSERT INTO sports (category_id, sport_name, sport_code, governing_body, is_olympic) VALUES
    (1, 'Football',         'FOOTB', 'FIFA',             TRUE),
    (1, 'Basketball',       'BSKTB', 'FIBA',             TRUE),
    (1, 'Volleyball',       'VOLLB', 'FIVB',             TRUE),
    (1, 'Handball',         'HNDBL', 'IHF',              TRUE),
    (1, 'Rugby',            'RUGBY', 'World Rugby',      TRUE),
    (2, 'Cycling',          'CYCLE', 'UCI',              TRUE),
    (2, 'Triathlon',        'TRIAT', 'World Triathlon',  TRUE),
    (2, 'Weightlifting',    'WLIFT', 'IWF',              TRUE),
    (3, 'Boxing',           'BOXNG', 'AIBA',             TRUE),
    (3, 'Judo',             'JUDOO', 'IJF',              TRUE),
    (3, 'Taekwondo',        'TAEKW', 'WT',               TRUE),
    (3, 'Wrestling',        'WRSTL', 'UWW',              TRUE),
    (4, 'Swimming',         'SWIMM', 'FINA',             TRUE),
    (5, 'Tennis',           'TENNS', 'ITF',              TRUE),
    (5, 'Badminton',        'BADMN', 'BWF',              TRUE),
    (5, 'Table Tennis',     'TTNNS', 'ITTF',             TRUE),
    (6, 'Athletics',        'ATHLT', 'World Athletics',  TRUE);


-- ============================================================
-- SECTION 17: KEY STORED PROCEDURES & FUNCTIONS
-- ============================================================

-- Function: Automatically flag DOB mismatch when NIDA data is received
CREATE OR REPLACE FUNCTION fn_check_dob_mismatch(
    p_person_id     INT,
    p_nida_dob      DATE
) RETURNS VOID AS $$
DECLARE
    v_registered_dob DATE;
    v_diff_days      INT;
BEGIN
    SELECT date_of_birth INTO v_registered_dob
    FROM persons WHERE person_id = p_person_id;

    v_diff_days := ABS(p_nida_dob - v_registered_dob);

    UPDATE persons SET
        nida_dob_match        = (v_diff_days = 0),
        nida_mismatch_days    = v_diff_days,
        nida_verification_date = CURRENT_TIMESTAMP,
        nida_status = CASE
            WHEN v_diff_days = 0   THEN 'VERIFIED'
            WHEN v_diff_days <= 30 THEN 'MISMATCH'   -- Small discrepancy
            ELSE                        'FLAGGED'    -- Large discrepancy = likely fraud
        END,
        updated_at = CURRENT_TIMESTAMP
    WHERE person_id = p_person_id;

    -- Auto-create fraud investigation if large mismatch (>90 days)
    IF v_diff_days > 90 THEN
        INSERT INTO age_fraud_investigations (
            person_id, dob_on_file, dob_from_nida,
            dob_difference_days, evidence_type,
            evidence_notes, investigation_status
        ) VALUES (
            p_person_id, v_registered_dob, p_nida_dob,
            v_diff_days, 'NIDA_MISMATCH',
            'Auto-flagged: DOB difference of ' || v_diff_days || ' days detected via NIDA verification.',
            'OPEN'
        );
    END IF;

    -- Block eligibility if mismatch exists
    IF v_diff_days > 0 THEN
        UPDATE players SET
            eligible_to_play    = FALSE,
            registration_status = 'PENDING_VERIFICATION'
        WHERE person_id = p_person_id;
    ELSE
        UPDATE players SET
            eligible_to_play = TRUE
        WHERE person_id = p_person_id
          AND registration_status != 'SUSPENDED'
          AND registration_status != 'BANNED';
    END IF;
END;
$$ LANGUAGE plpgsql;


-- Function: Check if a player is eligible for a specific age category
CREATE OR REPLACE FUNCTION fn_is_player_eligible(
    p_player_id    INT,
    p_age_cat_id   INT,
    p_check_date   DATE DEFAULT CURRENT_DATE
) RETURNS TABLE (
    is_eligible     BOOLEAN,
    reason          TEXT
) AS $$
DECLARE
    v_dob       DATE;
    v_age       INT;
    v_min_age   INT;
    v_max_age   INT;
    v_nida_status VARCHAR(20);
    v_eligible  BOOLEAN;
    v_pl_status VARCHAR(20);
BEGIN
    SELECT p.date_of_birth, p.nida_status, pl.registration_status
    INTO v_dob, v_nida_status, v_pl_status
    FROM persons p
    JOIN players pl ON pl.person_id = p.person_id
    WHERE pl.player_id = p_player_id;

    SELECT min_age, max_age INTO v_min_age, v_max_age
    FROM age_categories WHERE age_cat_id = p_age_cat_id;

    v_age := EXTRACT(YEAR FROM AGE(p_check_date, v_dob));

    IF v_nida_status NOT IN ('VERIFIED') THEN
        RETURN QUERY SELECT FALSE, 'NIDA verification not completed or failed';
        RETURN;
    END IF;
    IF v_pl_status IN ('SUSPENDED','BANNED') THEN
        RETURN QUERY SELECT FALSE, 'Player is ' || v_pl_status;
        RETURN;
    END IF;
    IF v_age < v_min_age THEN
        RETURN QUERY SELECT FALSE, 'Player age (' || v_age || ') below minimum (' || v_min_age || ')';
        RETURN;
    END IF;
    IF v_max_age IS NOT NULL AND v_age > v_max_age THEN
        RETURN QUERY SELECT FALSE, 'Player age (' || v_age || ') exceeds maximum (' || v_max_age || ')';
        RETURN;
    END IF;

    RETURN QUERY SELECT TRUE, 'Player is eligible';
END;
$$ LANGUAGE plpgsql;


-- Trigger: Auto-update updated_at timestamp
CREATE OR REPLACE FUNCTION fn_update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_persons_updated
    BEFORE UPDATE ON persons
    FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

CREATE TRIGGER trg_players_updated
    BEFORE UPDATE ON players
    FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

CREATE TRIGGER trg_clubs_updated
    BEFORE UPDATE ON clubs
    FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

CREATE TRIGGER trg_matches_updated
    BEFORE UPDATE ON matches
    FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

-- ============================================================
-- END OF SCHEMA
-- ============================================================
