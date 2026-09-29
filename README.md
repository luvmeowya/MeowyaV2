#include <Windows.h>
#include <TlHelp32.h>
#include <string>
#include <vector>
#include <cmath>
#include <thread>
#include <atomic>
#include <cstdio>
#include <unordered_map>
#include <algorithm>

#pragma comment(linker, "/SUBSYSTEM:WINDOWS")

namespace cfg {
    std::atomic<bool> esp_enabled{ false };
    std::atomic<bool> esp_box{ false };
    std::atomic<bool> esp_corner{ false };
    std::atomic<bool> esp_filled{ false };
    std::atomic<bool> esp_outline{ false };
    std::atomic<bool> esp_head{ false };
    std::atomic<bool> esp_name{ false };
    std::atomic<bool> esp_tracer{ false };
    std::atomic<bool> esp_china_hat{ false };
    std::atomic<bool> esp_skeleton{ false };
    std::atomic<bool> esp_skel_lock{ false };
    std::atomic<bool> esp_healthbar{ false };
    std::atomic<bool> esp_rainbow{ false };
    std::atomic<bool> esp_fov_only{ false };
    std::atomic<bool> esp_offscreen{ false };
    std::atomic<bool> esp_hp_color{ false };
    std::atomic<bool> esp_lock_hl{ true };
    std::atomic<bool> esp_lock_only{ false };
    std::atomic<bool> watermark{ true };
    std::atomic<bool> keylist{ true };
    std::atomic<bool> crosshair{ false };
    std::atomic<bool> crosshair_dot{ false };
    std::atomic<bool> aim_line{ false };
    std::atomic<bool> radar_enabled{ false };
    std::atomic<bool> radar_rings{ false };
    std::atomic<bool> bullet_tracers{ false };
    std::atomic<bool> lock_info{ true };
    std::atomic<bool> pred_mark{ true };
    std::atomic<bool> fov_on_aim{ false };
    std::atomic<bool> aimbot{ false };
    std::atomic<bool> silent_aim{ false };
    std::atomic<bool> aim_sticky{ true };
    std::atomic<bool> aim_beep{ false };
    std::atomic<bool> aim_lowhp{ false };
    std::atomic<bool> aim_closest{ false };
    std::atomic<bool> aim_body{ false };
    std::atomic<bool> aim_multipoint{ false };
    std::atomic<bool> aim_vis_only{ false };
    std::atomic<bool> aim_flick{ false };
    std::atomic<bool> aim_front{ false };
    std::atomic<bool> aim_on_shot{ false };
    std::atomic<bool> triggerbot{ false };
    std::atomic<bool> hitmarker{ true };
    std::atomic<bool> hit_sound{ true };
    std::atomic<bool> wall_check{ false };
    std::atomic<bool> skip_dead{ false };
    std::atomic<bool> team_check{ false };
    std::atomic<bool> nearby_alert{ false };
    std::atomic<bool> danger_banner{ true };
    std::atomic<bool> auto_bhop{ false };
    std::atomic<bool> auto_sprint{ false };
    std::atomic<bool> anti_afk{ false };
    std::atomic<bool> show_vel{ false };
    std::atomic<bool> hide_console{ false };
    std::atomic<bool> stream_mode{ false };
    std::atomic<bool> menu_topmost{ true };
    std::atomic<bool> flip_y{ false };
    std::atomic<bool> use_mat_330{ false };
    std::atomic<bool> use_formula2{ false };
    std::atomic<bool> use_cam_w2s{ true };
    std::atomic<bool> panic{ false };
    std::atomic<int> tracer_pos{ 2 };
    std::atomic<int> line_thick{ 1 };
    std::atomic<int> esp_max{ 32 };
    std::atomic<float> aim_fov_pct{ 15.f };
    std::atomic<float> aim_smooth{ 1.f };
    std::atomic<float> aim_lead{ 0.f };
    std::atomic<int> max_dist{ 2000 };
    std::atomic<int> aim_max_dist{ 2000 };
    std::atomic<int> trigger_delay{ 40 };
    std::atomic<int> nearby_studs{ 50 };
    std::atomic<int> col_r{ 0 }, col_g{ 255 }, col_b{ 100 };
    std::atomic<int> col_hr{ 255 }, col_hg{ 60 }, col_hb{ 60 };
    std::atomic<int> bt_r{ 255 }, bt_g{ 200 }, bt_b{ 50 };
}

HANDLE g_h = nullptr;
std::atomic<bool> g_running{ true };
std::string g_status = "starting...";
std::string g_localName = "-";
HWND g_menu = nullptr;
bool g_menuVisible = true;

struct Vec3 {
    float x = 0, y = 0, z = 0;
    float Dist(const Vec3& o) const {
        float dx = x - o.x, dy = y - o.y, dz = z - o.z;
        return sqrtf(dx * dx + dy * dy + dz * dz);
    }
};
struct Vec2 { float x = 0, y = 0; };
struct BoneScr { bool ok = false; Vec2 s{}; };

struct Entity {
    std::string name;
    Vec3 head, root, vel{};
    float dist = 0, hp = -1.f;
    bool valid = false, rootValid = false, visible = false, inFront = true;
    Vec2 screenHead{}, screenRoot{}, screenAim{}, screenPred{};
    bool predOk = false;
    float boxH = 40, boxW = 20;
    BoneScr bHead, bUpper, bLower, bHrp;
    BoneScr bLUA, bLLA, bLH, bRUA, bRLA, bRH;
    BoneScr bLUL, bLLL, bLF, bRUL, bRLL, bRF;
    BoneScr bLArm, bRArm, bLLeg, bRLeg;
    bool isR15 = false;
};

struct BulletTrace {
    Vec2 from{}, to{};
    DWORD expire = 0;
};

std::vector<Entity> g_entities;
std::vector<BulletTrace> g_traces;
CRITICAL_SECTION g_cs;
Vec3 g_localPos{}, g_localVel{};
uintptr_t g_localTeam = 0;
float g_view[16]{};
float g_gameW = 1280, g_gameH = 720;
int g_winX = 0, g_winY = 0, g_winW = 1280, g_winH = 720;
int g_radarAx = 0, g_radarAy = 2;
std::string g_aimLock;
Vec2 g_lockScreen{}, g_lockPred{};
float g_lockHp = -1.f, g_lockDist = 0.f;
bool g_lockValid = false, g_lockPredOk = false;
int g_playerCount = 0;
float g_fps = 0.f, g_closestEnemy = 99999.f;
DWORD g_hitmarkerUntil = 0;

Vec3 g_camPos{};
float g_camRight[3]{ 1,0,0 }, g_camUp[3]{ 0,1,0 }, g_camLook[3]{ 0,0,-1 };
float g_camFov = 70.f;
bool g_camOk = false;
int g_camLayout = 0;

struct Track { Vec3 last; DWORD ms; };
std::unordered_map<std::string, Track> g_track;

struct PartCache {
    uintptr_t character = 0, hrp = 0, head = 0;
    uintptr_t upperTorso = 0, lowerTorso = 0, torso = 0;
    uintptr_t lUpperArm = 0, lLowerArm = 0, lHand = 0;
    uintptr_t rUpperArm = 0, rLowerArm = 0, rHand = 0;
    uintptr_t lUpperLeg = 0, lLowerLeg = 0, lFoot = 0;
    uintptr_t rUpperLeg = 0, rLowerLeg = 0, rFoot = 0;
    uintptr_t lArm = 0, rArm = 0, lLeg = 0, rLeg = 0;
    DWORD lastMs = 0;
};
std::unordered_map<std::string, PartCache> g_partCache;

float GetAimFovPx() { return (g_winH * cfg::aim_fov_pct.load()) / 100.f; }
int Thick() { return max(1, cfg::line_thick.load()); }

void AddTrace(Vec2 from, Vec2 to) {
    if (!cfg::bullet_tracers) return;
    BulletTrace t;
    t.from = from; t.to = to;
    t.expire = GetTickCount() + 450;
    EnterCriticalSection(&g_cs);
    g_traces.push_back(t);
    if (g_traces.size() > 24) g_traces.erase(g_traces.begin());
    LeaveCriticalSection(&g_cs);
}

void PlayHitSound() {
    if (!cfg::hit_sound) return;
    Beep(1200, 30);
    Beep(900, 20);
}

DWORD GetPid(const wchar_t* name) {
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    PROCESSENTRY32W pe{ sizeof(pe) };
    DWORD pid = 0;
    if (Process32FirstW(snap, &pe)) {
        do {
            if (_wcsicmp(pe.szExeFile, name) == 0) { pid = pe.th32ProcessID; break; }
        } while (Process32NextW(snap, &pe));
    }
    CloseHandle(snap);
    return pid;
}

DWORD FindRobloxPid() {
    const wchar_t* names[] = {
        L"RobloxPlayerBeta.exe", L"RobloxPlayer.exe",
        L"Windows10Universal.exe", L"Roblox.exe"
    };
    for (auto n : names) {
        DWORD p = GetPid(n);
        if (p) return p;
    }
    return 0;
}

uintptr_t GetModuleBase(DWORD pid, const wchar_t* mod) {
    HANDLE snap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE | TH32CS_SNAPMODULE32, pid);
    if (snap == INVALID_HANDLE_VALUE) return 0;
    MODULEENTRY32W me{ sizeof(me) };
    uintptr_t base = 0;
    if (Module32FirstW(snap, &me)) {
        do {
            if (_wcsicmp(me.szModule, mod) == 0) {
                base = (uintptr_t)me.modBaseAddr;
                break;
            }
        } while (Module32NextW(snap, &me));
    }
    CloseHandle(snap);
    return base;
}

template<typename T>
T Read(uintptr_t addr) {
    T v{};
    if (g_h) ReadProcessMemory(g_h, (LPCVOID)addr, &v, sizeof(T), nullptr);
    return v;
}

std::string ReadString(uintptr_t addr) {
    int32_t len = Read<int32_t>(addr + 0x10);
    if (len <= 0 || len > 200) return "";
    if (len >= 16) {
        uintptr_t ptr = Read<uintptr_t>(addr);
        char buf[128]{};
        ReadProcessMemory(g_h, (LPCVOID)ptr, buf, min(len, 127), nullptr);
        return std::string(buf);
    }
    char buf[16]{};
    ReadProcessMemory(g_h, (LPCVOID)addr, buf, 16, nullptr);
    return std::string(buf);
}

namespace off {
    constexpr uintptr_t VisualEnginePointer = 0x851BF08;
    constexpr uintptr_t VisualEngine_FakeDM = 0xAF0;
    constexpr uintptr_t VisualEngine_ViewMatrix = 0x1B0;
    constexpr uintptr_t VisualEngine_ViewMatrixAlt = 0x330;
    constexpr uintptr_t VisualEngine_Dimensions = 0xB10;
    constexpr uintptr_t FakeDataModelPointer = 0x8EE1728;
    constexpr uintptr_t FakeDataModel_Real = 0x1F8;
    constexpr uintptr_t Instance_NameContainer = 0x70;
    constexpr uintptr_t Instance_Name = 0x8;
    constexpr uintptr_t Instance_ChildrenStart = 0x78;
    constexpr uintptr_t Instance_ChildrenEnd = 0x8;
    constexpr uintptr_t Player_LocalPlayer = 0x120;
    constexpr uintptr_t Player_ModelInstance = 0x288;
    constexpr uintptr_t Player_Team = 0x2C8;
    constexpr uintptr_t BasePart_Primitive = 0x178;
    constexpr uintptr_t Primitive_Position = 0xD4;
    constexpr uintptr_t Primitive_Velocity = 0xE0;
    constexpr uintptr_t Workspace_CurrentCamera = 0x4A8;
    constexpr uintptr_t Cam_Rotation = 0xC8;
    constexpr uintptr_t Cam_Position = 0xEC;
    constexpr uintptr_t Cam_FOV = 0x130;
}

std::vector<uintptr_t> GetChildren(uintptr_t inst) {
    std::vector<uintptr_t> out;
    uintptr_t startPtr = Read<uintptr_t>(inst + off::Instance_ChildrenStart);
    if (!startPtr) return out;
    uintptr_t begin = Read<uintptr_t>(startPtr);
    uintptr_t end = Read<uintptr_t>(startPtr + off::Instance_ChildrenEnd);
    if (!begin || !end || end <= begin || (end - begin) > 0x4000) return out;
    for (uintptr_t p = begin; p < end; p += 0x10) {
        uintptr_t child = Read<uintptr_t>(p);
        if (child) out.push_back(child);
    }
    return out;
}

std::string GetName(uintptr_t inst) {
    uintptr_t nameCont = Read<uintptr_t>(inst + off::Instance_NameContainer);
    if (!nameCont) return "";
    return ReadString(nameCont + off::Instance_Name);
}

uintptr_t FindChild(uintptr_t parent, const std::string& name) {
    for (auto c : GetChildren(parent))
        if (GetName(c) == name) return c;
    return 0;
}

void FindParts(uintptr_t character, PartCache& pc) {
    pc.hrp = pc.head = 0;
    pc.upperTorso = pc.lowerTorso = pc.torso = 0;
    pc.lUpperArm = pc.lLowerArm = pc.lHand = 0;
    pc.rUpperArm = pc.rLowerArm = pc.rHand = 0;
    pc.lUpperLeg = pc.lLowerLeg = pc.lFoot = 0;
    pc.rUpperLeg = pc.rLowerLeg = pc.rFoot = 0;
    pc.lArm = pc.rArm = pc.lLeg = pc.rLeg = 0;
    for (auto c : GetChildren(character)) {
        std::string n = GetName(c);
        if (n == "HumanoidRootPart") pc.hrp = c;
        else if (n == "Head") pc.head = c;
        else if (n == "UpperTorso") pc.upperTorso = c;
        else if (n == "LowerTorso") pc.lowerTorso = c;
        else if (n == "Torso") pc.torso = c;
        else if (n == "LeftUpperArm") pc.lUpperArm = c;
        else if (n == "LeftLowerArm") pc.lLowerArm = c;
        else if (n == "LeftHand") pc.lHand = c;
        else if (n == "RightUpperArm") pc.rUpperArm = c;
        else if (n == "RightLowerArm") pc.rLowerArm = c;
        else if (n == "RightHand") pc.rHand = c;
        else if (n == "LeftUpperLeg") pc.lUpperLeg = c;
        else if (n == "LeftLowerLeg") pc.lLowerLeg = c;
        else if (n == "LeftFoot") pc.lFoot = c;
        else if (n == "RightUpperLeg") pc.rUpperLeg = c;
        else if (n == "RightLowerLeg") pc.rLowerLeg = c;
        else if (n == "RightFoot") pc.rFoot = c;
        else if (n == "Left Arm") pc.lArm = c;
        else if (n == "Right Arm") pc.rArm = c;
        else if (n == "Left Leg") pc.lLeg = c;
        else if (n == "Right Leg") pc.rLeg = c;
    }
    if (!pc.hrp) {
        if (pc.upperTorso) pc.hrp = pc.upperTorso;
        else if (pc.torso) pc.hrp = pc.torso;
        else if (pc.head) pc.hrp = pc.head;
    }
}

Vec3 GetPartPos(uintptr_t part) {
    if (!part) return {};
    uintptr_t prim = Read<uintptr_t>(part + off::BasePart_Primitive);
    if (!prim) return {};
    return Read<Vec3>(prim + off::Primitive_Position);
}

Vec3 GetPartVel(uintptr_t part) {
    if (!part) return {};
    uintptr_t prim = Read<uintptr_t>(part + off::BasePart_Primitive);
    if (!prim) return {};
    return Read<Vec3>(prim + off::Primitive_Velocity);
}

bool RobloxFocused() {
    HWND fg = GetForegroundWindow();
    if (!fg) return false;
    char title[256]{};
    GetWindowTextA(fg, title, 255);
    return strstr(title, "Roblox") || strstr(title, "roblox");
}

void TapSpace() {
    if (!RobloxFocused()) return;
    keybd_event(VK_SPACE, 0x39, 0, 0);
    keybd_event(VK_SPACE, 0x39, KEYEVENTF_KEYUP, 0);
}

void HoldShift(bool down) {
    if (!RobloxFocused()) return;
    if (down) keybd_event(VK_SHIFT, 0x2A, 0, 0);
    else keybd_event(VK_SHIFT, 0x2A, KEYEVENTF_KEYUP, 0);
}

void ClickMouse() {
    if (!RobloxFocused()) return;
    mouse_event(MOUSEEVENTF_LEFTDOWN, 0, 0, 0, 0);
    mouse_event(MOUSEEVENTF_LEFTUP, 0, 0, 0, 0);
}

float ReadHealth(uintptr_t character) {
    uintptr_t hum = FindChild(character, "Humanoid");
    if (!hum) return -1.f;
    const uintptr_t candidates[] = { 0x180, 0x184, 0x198, 0x19C, 0x17C };
    for (auto o : candidates) {
        float hp = Read<float>(hum + o);
        if (hp > 0.f && hp <= 10000.f) return hp;
        if (hp == 0.f) return 0.f;
    }
    return -1.f;
}

void UpdateRobloxWindow() {
    HWND wnd = FindWindowA(nullptr, "Roblox");
    if (!wnd) {
        g_winX = 0; g_winY = 0;
        g_winW = GetSystemMetrics(SM_CXSCREEN);
        g_winH = GetSystemMetrics(SM_CYSCREEN);
        return;
    }
    RECT c{};
    GetClientRect(wnd, &c);
    POINT tl{ 0, 0 };
    ClientToScreen(wnd, &tl);
    g_winX = tl.x; g_winY = tl.y;
    g_winW = max(1, (int)(c.right - c.left));
    g_winH = max(1, (int)(c.bottom - c.top));
}

uintptr_t ResolveDataModel(uintptr_t base, uintptr_t& outVE) {
    outVE = Read<uintptr_t>(base + off::VisualEnginePointer);
    if (outVE) {
        uintptr_t fake = Read<uintptr_t>(outVE + off::VisualEngine_FakeDM);
        if (fake) {
            uintptr_t dm = Read<uintptr_t>(fake + off::FakeDataModel_Real);
            if (dm) return dm;
        }
    }
    uintptr_t fake2 = Read<uintptr_t>(base + off::FakeDataModelPointer);
    if (fake2) {
        uintptr_t dm = Read<uintptr_t>(fake2 + off::FakeDataModel_Real);
        if (dm) return dm;
    }
    return 0;
}

void UpdateCamera(uintptr_t dm) {
    g_camOk = false;
    uintptr_t ws = FindChild(dm, "Workspace");
    if (!ws) return;
    uintptr_t cam = Read<uintptr_t>(ws + off::Workspace_CurrentCamera);
    if (!cam) cam = FindChild(ws, "CurrentCamera");
    if (!cam) return;
    const uintptr_t posOffs[] = { 0xEC, 0xF0, 0xE4, 0x100 };
    g_camPos = {};
    for (auto o : posOffs) {
        Vec3 p = Read<Vec3>(cam + o);
        if (fabsf(p.x) < 1e6f && fabsf(p.y) < 1e6f && fabsf(p.z) < 1e6f &&
            (fabsf(p.x) + fabsf(p.y) + fabsf(p.z)) > 1.f) {
            g_camPos = p;
            break;
        }
    }
    float r[9]{};
    ReadProcessMemory(g_h, (LPCVOID)(cam + off::Cam_Rotation), r, 36, nullptr);
    g_camRight[0] = r[0]; g_camRight[1] = r[1]; g_camRight[2] = r[2];
    g_camUp[0] = r[3]; g_camUp[1] = r[4]; g_camUp[2] = r[5];
    g_camLook[0] = -r[6]; g_camLook[1] = -r[7]; g_camLook[2] = -r[8];
    float fov = Read<float>(cam + off::Cam_FOV);
    if (fov < 1.f || fov > 120.f) fov = Read<float>(cam + 0x140);
    g_camFov = (fov > 1.f && fov < 120.f) ? fov : 70.f;
    g_camOk = true;
}

float Comp(const Vec3& p, int i) {
    return i == 0 ? p.x : (i == 1 ? p.y : p.z);
}

void PickRadarAxes(const std::vector<Entity>& ents) {
    float minA[3] = { 1e9f, 1e9f, 1e9f };
    float maxA[3] = { -1e9f, -1e9f, -1e9f };
    auto upd = [&](const Vec3& p) {
        float a[3] = { p.x, p.y, p.z };
        for (int i = 0; i < 3; i++) {
            if (a[i] < minA[i]) minA[i] = a[i];
            if (a[i] > maxA[i]) maxA[i] = a[i];
        }
        };
    upd(g_localPos);
    for (auto& e : ents) upd(e.root);
    float span[3] = { maxA[0] - minA[0], maxA[1] - minA[1], maxA[2] - minA[2] };
    if (span[0] >= span[1] && span[0] >= span[2]) {
        g_radarAx = 0; g_radarAy = (span[1] >= span[2]) ? 1 : 2;
    }
    else if (span[1] >= span[0] && span[1] >= span[2]) {
        g_radarAx = 1; g_radarAy = (span[0] >= span[2]) ? 0 : 2;
    }
    else {
        g_radarAx = 2; g_radarAy = (span[0] >= span[1]) ? 0 : 1;
    }
}

bool WorldToScreenMatrix(const Vec3& w, Vec2& s) {
    float* m = g_view;
    float x, y, ww;
    if (cfg::use_formula2) {
        x = w.x * m[0] + w.y * m[1] + w.z * m[2] + m[3];
        y = w.x * m[4] + w.y * m[5] + w.z * m[6] + m[7];
        ww = w.x * m[8] + w.y * m[9] + w.z * m[10] + m[11];
    }
    else {
        x = w.x * m[0] + w.y * m[1] + w.z * m[2] + m[3];
        y = w.x * m[4] + w.y * m[5] + w.z * m[6] + m[7];
        ww = w.x * m[12] + w.y * m[13] + w.z * m[14] + m[15];
    }
    if (ww < 0.08f) return false;
    float inv = 1.f / ww;
    float gx = g_gameW * 0.5f * (1.f + x * inv);
    float gy = cfg::flip_y ? g_gameH * 0.5f * (1.f + y * inv) : g_gameH * 0.5f * (1.f - y * inv);
    s.x = (float)g_winX + gx / max(g_gameW, 1.f) * (float)g_winW;
    s.y = (float)g_winY + gy / max(g_gameH, 1.f) * (float)g_winH;
    return true;
}

bool WorldToScreenCam(const Vec3& w, Vec2& s) {
    if (!g_camOk) return false;
    float dx = w.x - g_camPos.x, dy = w.y - g_camPos.y, dz = w.z - g_camPos.z;
    float rx = g_camRight[0], ry = g_camRight[1], rz = g_camRight[2];
    float ux = g_camUp[0], uy = g_camUp[1], uz = g_camUp[2];
    float lx = g_camLook[0], ly = g_camLook[1], lz_v = g_camLook[2];
    if (g_camLayout == 1) { lx = -lx; ly = -ly; lz_v = -lz_v; }
    if (g_camLayout == 2) { ux = -ux; uy = -uy; uz = -uz; }
    float localX = dx * rx + dy * ry + dz * rz;
    float localY = dx * ux + dy * uy + dz * uz;
    float localZ = dx * lx + dy * ly + dz * lz_v;
    if (localZ < 0.2f) return false;
    float fovRad = g_camFov * 0.01745329252f;
    float th = tanf(fovRad * 0.5f);
    if (th < 0.05f) th = 0.05f;
    float aspect = (float)g_winW / max(1.f, (float)g_winH);
    float nx = (localX / localZ) / (th * aspect);
    float ny = (localY / localZ) / th;
    float sx = (nx + 1.f) * 0.5f * (float)g_winW;
    float sy = cfg::flip_y ? (ny + 1.f) * 0.5f * (float)g_winH : (1.f - ny) * 0.5f * (float)g_winH;
    s.x = (float)g_winX + sx;
    s.y = (float)g_winY + sy;
    return true;
}

bool WorldToScreen(const Vec3& w, Vec2& s) {
    if (cfg::use_cam_w2s && g_camOk) {
        if (WorldToScreenCam(w, s)) return true;
    }
    return WorldToScreenMatrix(w, s);
}

bool OnScreen(const Vec2& s) {
    return s.x >= (float)g_winX && s.x <= (float)(g_winX + g_winW) &&
        s.y >= (float)g_winY && s.y <= (float)(g_winY + g_winH);
}

bool IsInFront(const Vec3& world) {
    if (!g_camOk) return true;
    float dx = world.x - g_camPos.x;
    float dy = world.y - g_camPos.y;
    float dz = world.z - g_camPos.z;
    return (dx * g_camLook[0] + dy * g_camLook[1] + dz * g_camLook[2]) > 0.05f;
}

BoneScr BoneToScr(uintptr_t part) {
    BoneScr b{};
    if (!part) return b;
    b.ok = WorldToScreen(GetPartPos(part), b.s);
    return b;
}

void UpdateVel(Entity& e) {
    DWORD now = GetTickCount();
    auto it = g_track.find(e.name);
    if (it != g_track.end()) {
        float dt = (now - it->second.ms) / 1000.f;
        if (dt > 0.001f && dt < 0.4f) {
            e.vel.x = (e.head.x - it->second.last.x) / dt;
            e.vel.y = (e.head.y - it->second.last.y) / dt;
            e.vel.z = (e.head.z - it->second.last.z) / dt;
        }
    }
    g_track[e.name] = { e.head, now };
}

COLORREF Rainbow(float t) {
    return RGB(
        (int)((0.5f + 0.5f * sinf(t)) * 255),
        (int)((0.5f + 0.5f * sinf(t + 2.094f)) * 255),
        (int)((0.5f + 0.5f * sinf(t + 4.188f)) * 255));
}

COLORREF HpColor(float hp) {
    float pct = hp / 100.f;
    if (pct > 1.f) pct = 1.f;
    if (pct < 0.f) pct = 0.f;
    return RGB((int)((1.f - pct) * 255), (int)(pct * 255), 40);
}

void Worker() {
    DWORD lastPid = 0;
    uintptr_t cachedBase = 0, cachedVE = 0, cachedDM = 0;
    int resolveCooldown = 0, badCam = 0;
    DWORD lastBhopMs = 0, lastTrigMs = 0, lastAfkMs = 0, lastNearMs = 0;
    std::string lastLock;
    DWORD fpsT0 = GetTickCount();
    int fpsFrames = 0;
    bool shiftHeld = false, lmbWas = false;

    while (g_running) {
        fpsFrames++;
        DWORD nowF = GetTickCount();
        if (nowF - fpsT0 >= 500) {
            g_fps = fpsFrames * 1000.f / (nowF - fpsT0);
            fpsFrames = 0; fpsT0 = nowF;
        }

        static bool insertWas = false, homeWas = false, endWas = false;
        static bool f1Was = false, f2Was = false, f3Was = false, f4Was = false;
        bool insertNow = (GetAsyncKeyState(VK_INSERT) & 0x8000) != 0;
        bool homeNow = (GetAsyncKeyState(VK_HOME) & 0x8000) != 0;
        bool endNow = (GetAsyncKeyState(VK_END) & 0x8000) != 0;
        bool f1 = (GetAsyncKeyState(VK_F1) & 0x8000) != 0;
        bool f2 = (GetAsyncKeyState(VK_F2) & 0x8000) != 0;
        bool f3 = (GetAsyncKeyState(VK_F3) & 0x8000) != 0;
        bool f4 = (GetAsyncKeyState(VK_F4) & 0x8000) != 0;

        if (insertNow && !insertWas && g_menu) {
            g_menuVisible = !g_menuVisible;
            ShowWindow(g_menu, g_menuVisible ? SW_SHOW : SW_HIDE);
            if (g_menuVisible && cfg::menu_topmost)
                SetWindowPos(g_menu, HWND_TOPMOST, 0, 0, 0, 0, SWP_NOMOVE | SWP_NOSIZE);
        }
        if (homeNow && !homeWas) cfg::panic = !cfg::panic.load();
        if (endNow && !endWas) { g_running = false; PostQuitMessage(0); }
        if (f1 && !f1Was) cfg::esp_enabled = !cfg::esp_enabled.load();
        if (f2 && !f2Was) cfg::aimbot = !cfg::aimbot.load();
        if (f3 && !f3Was) cfg::radar_enabled = !cfg::radar_enabled.load();
        if (f4 && !f4Was) cfg::triggerbot = !cfg::triggerbot.load();
        insertWas = insertNow; homeWas = homeNow; endWas = endNow;
        f1Was = f1; f2Was = f2; f3Was = f3; f4Was = f4;

        if (cfg::hide_console) {
            HWND c = GetConsoleWindow();
            if (c) ShowWindow(c, SW_HIDE);
        }

        if (cfg::anti_afk && RobloxFocused()) {
            DWORD now = GetTickCount();
            if (now - lastAfkMs > 30000) {
                mouse_event(MOUSEEVENTF_MOVE, 1, 0, 0, 0);
                mouse_event(MOUSEEVENTF_MOVE, (DWORD)-1, 0, 0, 0);
                lastAfkMs = now;
            }
        }

        DWORD pid = FindRobloxPid();
        if (!pid) {
            g_status = "waiting for Roblox...";
            g_localName = "-"; g_playerCount = 0;
            if (g_h) { CloseHandle(g_h); g_h = nullptr; }
            lastPid = 0; cachedBase = cachedVE = cachedDM = 0;
            g_track.clear(); g_partCache.clear(); g_aimLock.clear();
            if (shiftHeld) { HoldShift(false); shiftHeld = false; }
            Sleep(400); continue;
        }

        if (pid != lastPid) {
            if (g_h) { CloseHandle(g_h); g_h = nullptr; }
            lastPid = pid; cachedBase = cachedVE = cachedDM = 0;
            g_track.clear(); g_partCache.clear(); g_aimLock.clear();
        }

        if (!g_h) {
            g_h = OpenProcess(PROCESS_VM_READ | PROCESS_QUERY_INFORMATION, FALSE, pid);
            if (!g_h) { g_status = "RUN AS ADMIN"; Sleep(1000); continue; }
        }

        static int winTick = 0;
        if ((winTick++ % 12) == 0) UpdateRobloxWindow();

        if (!cachedBase) {
            const wchar_t* mods[] = {
                L"RobloxPlayerBeta.exe", L"RobloxPlayer.exe",
                L"Windows10Universal.exe", L"Roblox.exe"
            };
            for (auto m : mods) {
                cachedBase = GetModuleBase(pid, m);
                if (cachedBase) break;
            }
            if (!cachedBase) { Sleep(150); continue; }
        }

        if (!cachedDM || resolveCooldown-- <= 0) {
            cachedDM = ResolveDataModel(cachedBase, cachedVE);
            resolveCooldown = 30;
            if (!cachedDM) { g_status = "join a game"; Sleep(250); continue; }
        }

        if (cachedVE) {
            uintptr_t matOff = cfg::use_mat_330 ? off::VisualEngine_ViewMatrixAlt : off::VisualEngine_ViewMatrix;
            ReadProcessMemory(g_h, (LPCVOID)(cachedVE + matOff), g_view, 64, nullptr);
            static int dimTick = 0;
            if ((dimTick++ % 25) == 0) {
                float dims[2]{};
                ReadProcessMemory(g_h, (LPCVOID)(cachedVE + off::VisualEngine_Dimensions), dims, 8, nullptr);
                if (dims[0] > 200) g_gameW = dims[0];
                if (dims[1] > 200) g_gameH = dims[1];
            }
        }

        UpdateCamera(cachedDM);
        uintptr_t playersSvc = FindChild(cachedDM, "Players");
        if (!playersSvc) { Sleep(30); continue; }

        uintptr_t localPlayer = Read<uintptr_t>(playersSvc + off::Player_LocalPlayer);
        g_localName = localPlayer ? GetName(localPlayer) : "?";
        g_localTeam = localPlayer ? Read<uintptr_t>(localPlayer + off::Player_Team) : 0;

        if (localPlayer) {
            uintptr_t lc = Read<uintptr_t>(localPlayer + off::Player_ModelInstance);
            if (lc) {
                auto& pc = g_partCache["__local__"];
                DWORD now = GetTickCount();
                if (pc.character != lc || now - pc.lastMs > 200) {
                    FindParts(lc, pc); pc.character = lc; pc.lastMs = now;
                }
                if (pc.hrp) {
                    g_localPos = GetPartPos(pc.hrp);
                    g_localVel = GetPartVel(pc.hrp);
                }
                if (cfg::auto_bhop && pc.hrp) {
                    Vec3 vel = GetPartVel(pc.hrp);
                    if (vel.y > -2.5f && vel.y < 8.f && (now - lastBhopMs) > 35) {
                        TapSpace(); lastBhopMs = now;
                    }
                }
            }
        }

        if (cfg::auto_sprint) {
            bool want = (GetAsyncKeyState('W') & 0x8000) != 0;
            if (want && !shiftHeld) { HoldShift(true); shiftHeld = true; }
            if (!want && shiftHeld) { HoldShift(false); shiftHeld = false; }
        }
        else if (shiftHeld) {
            HoldShift(false); shiftHeld = false;
        }

        float cx = g_winX + g_winW * 0.5f;
        float cy = g_winY + g_winH * 0.5f;
        float fovPx = GetAimFovPx();
        bool needHp = cfg::skip_dead || cfg::esp_name || cfg::esp_healthbar || cfg::aim_lowhp || cfg::esp_hp_color || cfg::lock_info;
        bool needSkel = cfg::esp_skeleton || cfg::esp_skel_lock;
        float maxD = (float)cfg::max_dist.load();
        float aimMaxD = (float)cfg::aim_max_dist.load();
        float lead = cfg::aim_lead.load();
        DWORD nowMs = GetTickCount();

        std::vector<Entity> ents;
        ents.reserve(24);
        int onScreen = 0;
        g_lockValid = false;
        g_lockPredOk = false;
        g_closestEnemy = 99999.f;

        for (auto p : GetChildren(playersSvc)) {
            if (p == localPlayer) continue;
            Entity e;
            e.name = GetName(p);
            if (!g_localName.empty() && g_localName != "?" && e.name == g_localName) continue;

            if (cfg::team_check && g_localTeam) {
                uintptr_t theirTeam = Read<uintptr_t>(p + off::Player_Team);
                if (theirTeam && theirTeam == g_localTeam) continue;
            }

            uintptr_t character = Read<uintptr_t>(p + off::Player_ModelInstance);
            if (!character) continue;

            if (needHp) {
                e.hp = ReadHealth(character);
                if (cfg::skip_dead && e.hp == 0.f) continue;
            }

            auto& pc = g_partCache[e.name];
            if (pc.character != character || nowMs - pc.lastMs > 150) {
                FindParts(character, pc); pc.character = character; pc.lastMs = nowMs;
            }
            if (!pc.hrp) continue;

            e.root = GetPartPos(pc.hrp);
            e.head = pc.head ? GetPartPos(pc.head) : e.root;
            e.dist = g_localPos.Dist(e.root);
            if (e.dist < 1.5f || e.dist > maxD) continue;
            if (e.dist < g_closestEnemy) g_closestEnemy = e.dist;

            e.inFront = IsInFront(e.head);
            UpdateVel(e);
            e.valid = WorldToScreen(e.head, e.screenHead);
            e.rootValid = WorldToScreen(e.root, e.screenRoot);

            Vec2 headScr = e.screenHead, bodyScr = e.screenRoot;
            if (cfg::aim_multipoint && e.valid && e.rootValid) {
                float dh = sqrtf((headScr.x - cx) * (headScr.x - cx) + (headScr.y - cy) * (headScr.y - cy));
                float db = sqrtf((bodyScr.x - cx) * (bodyScr.x - cx) + (bodyScr.y - cy) * (bodyScr.y - cy));
                e.screenAim = (dh <= db) ? headScr : bodyScr;
            }
            else if (cfg::aim_body && e.rootValid) {
                e.screenAim = bodyScr;
            }
            else {
                e.screenAim = headScr;
            }

            if (lead > 0.001f) {
                Vec3 aimW = cfg::aim_body ? e.root : e.head;
                aimW.x += e.vel.x * lead;
                aimW.y += e.vel.y * lead;
                aimW.z += e.vel.z * lead;
                e.predOk = WorldToScreen(aimW, e.screenPred);
                if (e.predOk) e.screenAim = e.screenPred;
            }

            e.visible = e.valid && OnScreen(e.screenHead);

            if (e.valid && e.rootValid) {
                float pixelH = fabsf(e.screenRoot.y - e.screenHead.y);
                e.boxH = max(18.f, pixelH * 1.6f);
                e.boxW = e.boxH * 0.45f;
            }
            else {
                float d = max(e.dist, 1.f);
                e.boxH = max(16.f, min((g_winH * 0.5f) / d, (float)g_winH * 0.5f));
                e.boxW = e.boxH * 0.45f;
            }

            if (needSkel) {
                e.isR15 = (pc.upperTorso != 0);
                e.bHead = BoneToScr(pc.head); e.bHrp = BoneToScr(pc.hrp);
                if (e.isR15) {
                    e.bUpper = BoneToScr(pc.upperTorso);
                    e.bLower = BoneToScr(pc.lowerTorso ? pc.lowerTorso : pc.hrp);
                    e.bLUA = BoneToScr(pc.lUpperArm); e.bLLA = BoneToScr(pc.lLowerArm); e.bLH = BoneToScr(pc.lHand);
                    e.bRUA = BoneToScr(pc.rUpperArm); e.bRLA = BoneToScr(pc.rLowerArm); e.bRH = BoneToScr(pc.rHand);
                    e.bLUL = BoneToScr(pc.lUpperLeg); e.bLLL = BoneToScr(pc.lLowerLeg); e.bLF = BoneToScr(pc.lFoot);
                    e.bRUL = BoneToScr(pc.rUpperLeg); e.bRLL = BoneToScr(pc.rLowerLeg); e.bRF = BoneToScr(pc.rFoot);
                }
                else {
                    e.bUpper = BoneToScr(pc.torso ? pc.torso : pc.hrp);
                    e.bLArm = BoneToScr(pc.lArm); e.bRArm = BoneToScr(pc.rArm);
                    e.bLLeg = BoneToScr(pc.lLeg); e.bRLeg = BoneToScr(pc.rLeg);
                }
            }

            if (e.visible) onScreen++;
            ents.push_back(std::move(e));
        }

        // sort by screen FOV distance for esp_max
        std::sort(ents.begin(), ents.end(), [&](const Entity& a, const Entity& b) {
            float da = a.valid ? (a.screenHead.x - cx) * (a.screenHead.x - cx) + (a.screenHead.y - cy) * (a.screenHead.y - cy) : 1e12f;
            float db = b.valid ? (b.screenHead.x - cx) * (b.screenHead.x - cx) + (b.screenHead.y - cy) * (b.screenHead.y - cy) : 1e12f;
            return da < db;
            });

        g_playerCount = (int)ents.size();

        if (cfg::nearby_alert && g_closestEnemy < (float)cfg::nearby_studs.load()) {
            DWORD now = GetTickCount();
            if (now - lastNearMs > 1500) { Beep(600, 40); lastNearMs = now; }
        }

        if (cfg::use_cam_w2s && (int)ents.size() >= 2 && onScreen == 0) {
            if (++badCam > 35) { g_camLayout = (g_camLayout + 1) % 3; badCam = 0; }
        }
        else badCam = 0;

        auto CanAim = [&](const Entity& ent) -> bool {
            if (!ent.valid) return false;
            if (ent.dist > aimMaxD) return false;
            if (cfg::wall_check && !ent.visible) return false;
            if (cfg::aim_vis_only && !ent.visible) return false;
            if (cfg::aim_front && !ent.inFront) return false;
            return true;
            };

        bool rmb = (GetAsyncKeyState(VK_RBUTTON) & 0x8000) != 0;
        bool lmb = (GetAsyncKeyState(VK_LBUTTON) & 0x8000) != 0;
        if (!rmb) g_aimLock.clear();

        bool allowAim = cfg::aimbot && rmb;
        if (cfg::aim_on_shot) allowAim = allowAim && lmb;

        if (allowAim) {
            float best = cfg::aim_closest ? 1e9f : fovPx;
            float adx = 0, ady = 0;
            bool found = false;
            std::string pick;
            Vec2 lockScr{}, lockPred{};
            float bestHp = 1e9f, lockHp = -1.f, lockDist = 0.f;
            bool predOk = false;

            if (cfg::aim_sticky && !g_aimLock.empty()) {
                for (auto& e : ents) {
                    if (!CanAim(e) || e.name != g_aimLock) continue;
                    float dx = e.screenAim.x - cx, dy = e.screenAim.y - cy;
                    float ad = sqrtf(dx * dx + dy * dy);
                    if (ad < fovPx * 3.f || ad < 800.f) {
                        adx = dx; ady = dy; found = true; pick = e.name;
                        lockScr = e.screenAim; lockHp = e.hp; lockDist = e.dist;
                        predOk = e.predOk; lockPred = e.screenPred;
                    }
                    break;
                }
                if (!found) g_aimLock.clear();
            }

            if (!found) {
                for (auto& e : ents) {
                    if (!CanAim(e)) continue;
                    float dx = e.screenAim.x - cx, dy = e.screenAim.y - cy;
                    float ad = sqrtf(dx * dx + dy * dy);
                    if (cfg::aim_closest) {
                        if (e.dist > best) continue;
                        if (ad > fovPx) continue;
                        best = e.dist;
                    }
                    else {
                        if (ad > best) continue;
                        best = ad;
                    }
                    if (cfg::aim_lowhp && e.hp >= 0.f) {
                        if (e.hp > bestHp) continue;
                        bestHp = e.hp;
                    }
                    adx = dx; ady = dy; found = true; pick = e.name;
                    lockScr = e.screenAim; lockHp = e.hp; lockDist = e.dist;
                    predOk = e.predOk; lockPred = e.screenPred;
                }
                if (found) g_aimLock = pick;
            }

            if (found) {
                if (cfg::aim_beep && pick != lastLock) Beep(900, 30);
                lastLock = pick;
                g_lockScreen = lockScr; g_lockValid = true;
                g_lockHp = lockHp; g_lockDist = lockDist;
                g_lockPredOk = predOk; g_lockPred = lockPred;
                float sm = cfg::aim_flick ? 1.f : max(cfg::aim_smooth.load(), 1.f);
                float t = (sm <= 1.f) ? 1.f : (1.f / sm);
                float mx = adx * t, my = ady * t;
                if (fabsf(adx) < 8.f) mx = adx;
                if (fabsf(ady) < 8.f) my = ady;
                if (mx > 250.f) mx = 250.f; if (mx < -250.f) mx = -250.f;
                if (my > 250.f) my = 250.f; if (my < -250.f) my = -250.f;
                if ((int)mx != 0 || (int)my != 0)
                    mouse_event(MOUSEEVENTF_MOVE, (DWORD)(int)mx, (DWORD)(int)my, 0, 0);
            }
        }
        else lastLock.clear();

        if (cfg::silent_aim && lmb) {
            float best = fovPx, sdx = 0, sdy = 0;
            bool found = false;
            for (auto& e : ents) {
                if (!CanAim(e)) continue;
                float dx = e.screenAim.x - cx, dy = e.screenAim.y - cy;
                float ad = sqrtf(dx * dx + dy * dy);
                if (ad < best) { best = ad; sdx = dx; sdy = dy; found = true; }
            }
            if (found) {
                if (sdx > 200.f) sdx = 200.f; if (sdx < -200.f) sdx = -200.f;
                if (sdy > 200.f) sdy = 200.f; if (sdy < -200.f) sdy = -200.f;
                mouse_event(MOUSEEVENTF_MOVE, (DWORD)(int)sdx, (DWORD)(int)sdy, 0, 0);
            }
        }

        if (cfg::bullet_tracers && lmb && !lmbWas) {
            Vec2 from{ cx, cy };
            Vec2 to = g_lockValid ? g_lockScreen : Vec2{ cx, cy - 200.f };
            if (!g_lockValid) {
                float best = fovPx;
                for (auto& e : ents) {
                    if (!e.valid) continue;
                    float dx = e.screenAim.x - cx, dy = e.screenAim.y - cy;
                    float ad = sqrtf(dx * dx + dy * dy);
                    if (ad < best) { best = ad; to = e.screenAim; }
                }
            }
            AddTrace(from, to);
        }
        lmbWas = lmb;

        if (cfg::triggerbot) {
            float best = 12.f;
            bool onTarget = false;
            Vec2 hitScr{};
            for (auto& e : ents) {
                if (!CanAim(e)) continue;
                float dx = e.screenAim.x - cx, dy = e.screenAim.y - cy;
                if (sqrtf(dx * dx + dy * dy) < best) {
                    onTarget = true; hitScr = e.screenAim; break;
                }
            }
            DWORD now = GetTickCount();
            if (onTarget && (now - lastTrigMs) > (DWORD)cfg::trigger_delay.load()) {
                ClickMouse(); lastTrigMs = now;
                if (cfg::hitmarker) g_hitmarkerUntil = now + 180;
                PlayHitSound();
                if (cfg::bullet_tracers) AddTrace(Vec2{ cx, cy }, hitScr);
            }
        }

        static int axisTick = 0;
        if ((axisTick++ % 40) == 0) PickRadarAxes(ents);

        EnterCriticalSection(&g_cs);
        g_entities.swap(ents);
        LeaveCriticalSection(&g_cs);

        char st[240];
        sprintf_s(st, "p:%d on:%d fps:%.0f near:%.0f lock:%s%s",
            g_playerCount, onScreen, g_fps,
            g_closestEnemy > 90000.f ? 0.f : g_closestEnemy,
            g_aimLock.empty() ? "-" : g_aimLock.c_str(),
            cfg::panic.load() ? " [PANIC]" : "");
        g_status = st;
        Sleep(0);
    }
    if (shiftHeld) HoldShift(false);
    if (g_h) { CloseHandle(g_h); g_h = nullptr; }
}

HWND g_overlay = nullptr;

void Line(HDC hdc, int x1, int y1, int x2, int y2, COLORREF c, int t) {
    HPEN pen = CreatePen(PS_SOLID, t, c);
    HGDIOBJ o = SelectObject(hdc, pen);
    MoveToEx(hdc, x1, y1, nullptr); LineTo(hdc, x2, y2);
    SelectObject(hdc, o); DeleteObject(pen);
}
void Box(HDC hdc, int x, int y, int w, int h, COLORREF c, int t) {
    Line(hdc, x, y, x + w, y, c, t); Line(hdc, x + w, y, x + w, y + h, c, t);
    Line(hdc, x + w, y + h, x, y + h, c, t); Line(hdc, x, y + h, x, y, c, t);
}
void CornerBox(HDC hdc, int x, int y, int w, int h, COLORREF c, int t) {
    int len = max(4, w / 4);
    Line(hdc, x, y, x + len, y, c, t); Line(hdc, x, y, x, y + len, c, t);
    Line(hdc, x + w, y, x + w - len, y, c, t); Line(hdc, x + w, y, x + w, y + len, c, t);
    Line(hdc, x, y + h, x + len, y + h, c, t); Line(hdc, x, y + h, x, y + h - len, c, t);
    Line(hdc, x + w, y + h, x + w - len, y + h, c, t); Line(hdc, x + w, y + h, x + w, y + h - len, c, t);
}
void FilledBox(HDC hdc, int x, int y, int w, int h, COLORREF c) {
    HBRUSH br = CreateHatchBrush(HS_DIAGCROSS, c);
    RECT r{ x, y, x + w, y + h };
    FillRect(hdc, &r, br); DeleteObject(br);
    Box(hdc, x, y, w, h, c, Thick());
}
void FillRectC(HDC hdc, int x, int y, int w, int h, COLORREF c) {
    HBRUSH br = CreateSolidBrush(c);
    RECT r{ x, y, x + w, y + h };
    FillRect(hdc, &r, br); DeleteObject(br);
}
void FillCircle(HDC hdc, int cx, int cy, int r, COLORREF c) {
    HBRUSH br = CreateSolidBrush(c);
    HPEN pen = CreatePen(PS_SOLID, 1, c);
    HGDIOBJ o1 = SelectObject(hdc, pen), o2 = SelectObject(hdc, br);
    Ellipse(hdc, cx - r, cy - r, cx + r, cy + r);
    SelectObject(hdc, o1); SelectObject(hdc, o2);
    DeleteObject(pen); DeleteObject(br);
}
void ChinaHat(HDC hdc, float hx, float hy, float boxH, COLORREF c) {
    float apexY = hy - boxH * 0.55f;
    float radius = max(6.f, boxH * 0.28f);
    float baseY = hy - boxH * 0.05f;
    const int segs = 10; POINT pts[10];
    for (int i = 0; i < segs; i++) {
        float a = (6.2831853f * (float)i) / (float)segs;
        pts[i].x = (LONG)(hx + cosf(a) * radius);
        pts[i].y = (LONG)(baseY + sinf(a) * radius * 0.35f);
    }
    for (int i = 0; i < segs; i++) {
        int j = (i + 1) % segs;
        Line(hdc, pts[i].x, pts[i].y, pts[j].x, pts[j].y, c, 1);
    }
    int ax = (int)hx, ay = (int)apexY;
    for (int i = 0; i < segs; i++) Line(hdc, ax, ay, pts[i].x, pts[i].y, c, 1);
}
void BoneLine(HDC hdc, const BoneScr& a, const BoneScr& b, COLORREF c) {
    if (!a.ok || !b.ok) return;
    Line(hdc, (int)a.s.x, (int)a.s.y, (int)b.s.x, (int)b.s.y, c, Thick());
}
void DrawSkeleton(HDC hdc, const Entity& e, COLORREF c) {
    if (e.isR15) {
        BoneLine(hdc, e.bHead, e.bUpper, c);
        BoneLine(hdc, e.bUpper, e.bLower, c); BoneLine(hdc, e.bLower, e.bHrp, c);
        BoneLine(hdc, e.bUpper, e.bLUA, c); BoneLine(hdc, e.bLUA, e.bLLA, c); BoneLine(hdc, e.bLLA, e.bLH, c);
        BoneLine(hdc, e.bUpper, e.bRUA, c); BoneLine(hdc, e.bRUA, e.bRLA, c); BoneLine(hdc, e.bRLA, e.bRH, c);
        BoneLine(hdc, e.bLower, e.bLUL, c); BoneLine(hdc, e.bLUL, e.bLLL, c); BoneLine(hdc, e.bLLL, e.bLF, c);
        BoneLine(hdc, e.bLower, e.bRUL, c); BoneLine(hdc, e.bRUL, e.bRLL, c); BoneLine(hdc, e.bRLL, e.bRF, c);
    }
    else {
        BoneLine(hdc, e.bHead, e.bUpper, c);
        BoneLine(hdc, e.bUpper, e.bLArm, c); BoneLine(hdc, e.bUpper, e.bRArm, c);
        BoneLine(hdc, e.bUpper, e.bLLeg, c); BoneLine(hdc, e.bUpper, e.bRLeg, c);
        BoneLine(hdc, e.bUpper, e.bHrp, c);
    }
}
void HealthBar(HDC hdc, int boxX, int boxY, int boxH, float hp) {
    if (hp < 0.f) return;
    float pct = hp / 100.f;
    if (pct > 1.f) pct = 1.f; if (pct < 0.f) pct = 0.f;
    int barW = 3, barH = boxH, filled = (int)(barH * pct), x = boxX - 6;
    Box(hdc, x, boxY, barW, barH, RGB(0, 0, 0), 1);
    COLORREF hc = pct > 0.5f ? RGB(0, 220, 80) : (pct > 0.25f ? RGB(220, 200, 0) : RGB(220, 40, 40));
    for (int i = 0; i < filled; i++)
        Line(hdc, x + 1, boxY + barH - 1 - i, x + barW - 1, boxY + barH - 1 - i, hc, 1);
}
void OffscreenArrow(HDC hdc, float hx, float hy, float cx, float cy, COLORREF c) {
    float dx = hx - cx, dy = hy - cy;
    float len = sqrtf(dx * dx + dy * dy);
    if (len < 1.f) return;
    dx /= len; dy /= len;
    float edge = min((float)g_winW, (float)g_winH) * 0.42f;
    int ax = (int)(cx + dx * edge), ay = (int)(cy + dy * edge);
    float px = -dy, py = dx; int s = 10;
    POINT tri[3] = {
        { ax, ay },
        { ax - (int)(dx * s) + (int)(px * s * 0.5f), ay - (int)(dy * s) + (int)(py * s * 0.5f) },
        { ax - (int)(dx * s) - (int)(px * s * 0.5f), ay - (int)(dy * s) - (int)(py * s * 0.5f) }
    };
    HPEN pen = CreatePen(PS_SOLID, 1, c);
    HBRUSH br = CreateSolidBrush(c);
    HGDIOBJ o1 = SelectObject(hdc, pen), o2 = SelectObject(hdc, br);
    Polygon(hdc, tri, 3);
    SelectObject(hdc, o1); SelectObject(hdc, o2);
    DeleteObject(pen); DeleteObject(br);
}

void DrawLockInfo(HDC hdc) {
    if (!cfg::lock_info || !g_lockValid || g_aimLock.empty() || cfg::panic) return;
    int panelW = 170, panelH = 72;
    int px = g_winX + 14, py = g_winY + g_winH / 2 - panelH / 2;
    FillRectC(hdc, px, py, panelW, panelH, RGB(12, 12, 18));
    Box(hdc, px, py, panelW, panelH, RGB(0, 200, 120), 1);
    FillRectC(hdc, px + 8, py + 10, 40, 40, RGB(40, 40, 50));
    Box(hdc, px + 8, py + 10, 40, 40, RGB(80, 80, 90), 1);
    SetBkMode(hdc, TRANSPARENT);
    SetTextColor(hdc, RGB(140, 140, 150));
    TextOutA(hdc, px + 14, py + 22, "AV", 2);
    SetTextColor(hdc, RGB(255, 255, 255));
    char line[96];
    if (cfg::stream_mode) sprintf_s(line, "target locked");
    else sprintf_s(line, "%s", g_aimLock.c_str());
    TextOutA(hdc, px + 56, py + 10, line, (int)strlen(line));
    if (g_lockHp >= 0.f) sprintf_s(line, "HP  %.0f", g_lockHp);
    else sprintf_s(line, "HP  --");
    SetTextColor(hdc, g_lockHp >= 0.f && g_lockHp < 30.f ? RGB(255, 80, 80) : RGB(180, 255, 180));
    TextOutA(hdc, px + 56, py + 28, line, (int)strlen(line));
    sprintf_s(line, "DIST  %.0f", g_lockDist);
    SetTextColor(hdc, RGB(180, 200, 255));
    TextOutA(hdc, px + 56, py + 46, line, (int)strlen(line));
}

COLORREF EspColorVisible() { return RGB(cfg::col_r.load(), cfg::col_g.load(), cfg::col_b.load()); }
COLORREF EspColorHidden() { return RGB(cfg::col_hr.load(), cfg::col_hg.load(), cfg::col_hb.load()); }
COLORREF TracerColor() { return RGB(cfg::bt_r.load(), cfg::bt_g.load(), cfg::bt_b.load()); }

void DrawRadar(HDC hdc) {
    if (!cfg::radar_enabled || cfg::panic) return;
    const int size = 150, margin = 20;
    int rx = g_winX + g_winW - size - margin, ry = g_winY + margin;
    int cx = rx + size / 2, cy = ry + size / 2;
    float range = 350.f;
    HBRUSH bg = CreateSolidBrush(RGB(10, 10, 14));
    HPEN border = CreatePen(PS_SOLID, 2, RGB(0, 200, 110));
    HGDIOBJ o1 = SelectObject(hdc, border), o2 = SelectObject(hdc, bg);
    Ellipse(hdc, rx, ry, rx + size, ry + size);
    SelectObject(hdc, o1); SelectObject(hdc, o2);
    DeleteObject(bg); DeleteObject(border);
    if (cfg::radar_rings) {
        for (int ring = 1; ring <= 3; ring++) {
            int rr = (size / 2) * ring / 3;
            HPEN p = CreatePen(PS_SOLID, 1, RGB(30, 30, 40));
            HGDIOBJ o = SelectObject(hdc, p);
            SelectObject(hdc, GetStockObject(NULL_BRUSH));
            Ellipse(hdc, cx - rr, cy - rr, cx + rr, cy + rr);
            SelectObject(hdc, o); DeleteObject(p);
        }
    }
    Line(hdc, cx, ry + 6, cx, ry + size - 6, RGB(40, 40, 48), 1);
    Line(hdc, rx + 6, cy, rx + size - 6, cy, RGB(40, 40, 48), 1);
    FillCircle(hdc, cx, cy, 4, RGB(0, 255, 120));
    std::vector<Entity> ents;
    EnterCriticalSection(&g_cs); ents = g_entities; LeaveCriticalSection(&g_cs);
    for (auto& e : ents) {
        float dx = Comp(e.root, g_radarAx) - Comp(g_localPos, g_radarAx);
        float dy = Comp(e.root, g_radarAy) - Comp(g_localPos, g_radarAy);
        float dist = sqrtf(dx * dx + dy * dy);
        if (dist < 3.f || dist > range) continue;
        float scale = (size * 0.40f) / range;
        int px = cx + (int)(dx * scale), py = cy - (int)(dy * scale);
        int rmax = size / 2 - 8;
        if ((px - cx) * (px - cx) + (py - cy) * (py - cy) > rmax * rmax) continue;
        bool locked = (!g_aimLock.empty() && e.name == g_aimLock);
        FillCircle(hdc, px, py, locked ? 5 : 3, locked ? RGB(255, 255, 0) : RGB(255, 55, 55));
    }
}

void DrawEsp(HDC hdc) {
    float cx = g_winX + g_winW * 0.5f, cy = g_winY + g_winH * 0.5f;
    int t = Thick();
    float rainT = GetTickCount() * 0.003f;
    bool rmb = (GetAsyncKeyState(VK_RBUTTON) & 0x8000) != 0;

    if (cfg::watermark && !cfg::panic) {
        SetBkMode(hdc, TRANSPARENT);
        SetTextColor(hdc, RGB(0, 255, 140));
        char wm[200];
        float nearD = g_closestEnemy > 90000.f ? 0.f : g_closestEnemy;
        if (cfg::show_vel) {
            float spd = sqrtf(g_localVel.x * g_localVel.x + g_localVel.z * g_localVel.z);
            sprintf_s(wm, "meowya | %s | p:%d | fps:%.0f | spd:%.0f | near:%.0f",
                g_localName.c_str(), g_playerCount, g_fps, spd, nearD);
        }
        else {
            sprintf_s(wm, "meowya | %s | p:%d | fps:%.0f | near:%.0f",
                g_localName.c_str(), g_playerCount, g_fps, nearD);
        }
        TextOutA(hdc, g_winX + 12, g_winY + 10, wm, (int)strlen(wm));
    }

    if (cfg::keylist && !cfg::panic) {
        SetBkMode(hdc, TRANSPARENT);
        SetTextColor(hdc, RGB(180, 180, 190));
        TextOutA(hdc, g_winX + 12, g_winY + 28, "F1 ESP  F2 Aim  F3 Radar  F4 Trigger", 36);
        TextOutA(hdc, g_winX + 12, g_winY + 42, "INS Menu  HOME Panic  END Exit", 30);
    }

    if (cfg::danger_banner && !cfg::panic && g_closestEnemy < (float)cfg::nearby_studs.load()) {
        SetBkMode(hdc, TRANSPARENT);
        SetTextColor(hdc, RGB(255, 40, 40));
        char db[64];
        sprintf_s(db, "ENEMY NEAR  %.0f studs", g_closestEnemy);
        TextOutA(hdc, g_winX + g_winW / 2 - 80, g_winY + 60, db, (int)strlen(db));
    }

    if (cfg::bullet_tracers && !cfg::panic) {
        DWORD now = GetTickCount();
        EnterCriticalSection(&g_cs);
        auto traces = g_traces;
        std::vector<BulletTrace> keep;
        for (auto& tr : g_traces) if (now < tr.expire) keep.push_back(tr);
        g_traces = keep;
        LeaveCriticalSection(&g_cs);
        COLORREF tc = TracerColor();
        for (auto& tr : traces) {
            if (now >= tr.expire) continue;
            Line(hdc, (int)tr.from.x, (int)tr.from.y, (int)tr.to.x, (int)tr.to.y, tc, 2);
            FillCircle(hdc, (int)tr.to.x, (int)tr.to.y, 3, tc);
        }
    }

    DrawLockInfo(hdc);

    if (cfg::pred_mark && !cfg::panic && g_lockValid && g_lockPredOk && cfg::aim_lead.load() > 0.001f)
        FillCircle(hdc, (int)g_lockPred.x, (int)g_lockPred.y, 4, RGB(255, 180, 40));

    if (cfg::hitmarker && !cfg::panic && GetTickCount() < g_hitmarkerUntil) {
        int s = 10;
        Line(hdc, (int)cx - s, (int)cy - s, (int)cx + s, (int)cy + s, RGB(255, 255, 255), 2);
        Line(hdc, (int)cx + s, (int)cy - s, (int)cx - s, (int)cy + s, RGB(255, 255, 255), 2);
    }

    if (cfg::panic) return;

    if (cfg::crosshair) {
        int s = 8;
        Line(hdc, (int)cx - s, (int)cy, (int)cx + s, (int)cy, RGB(255, 255, 255), 1);
        Line(hdc, (int)cx, (int)cy - s, (int)cx, (int)cy + s, RGB(255, 255, 255), 1);
    }
    if (cfg::crosshair_dot) FillCircle(hdc, (int)cx, (int)cy, 2, RGB(255, 50, 50));

    if (cfg::aim_line && g_lockValid)
        Line(hdc, (int)cx, (int)cy, (int)g_lockScreen.x, (int)g_lockScreen.y, RGB(255, 80, 80), t);

    bool showFov = (cfg::aimbot || cfg::silent_aim || cfg::triggerbot);
    if (cfg::fov_on_aim) showFov = showFov && rmb;
    if (showFov) {
        float r = GetAimFovPx();
        HPEN pen = CreatePen(PS_SOLID, 1, RGB(200, 200, 200));
        HGDIOBJ o = SelectObject(hdc, pen);
        SelectObject(hdc, GetStockObject(NULL_BRUSH));
        Ellipse(hdc, (int)(cx - r), (int)(cy - r), (int)(cx + r), (int)(cy + r));
        SelectObject(hdc, o); DeleteObject(pen);
    }

    if (!cfg::esp_enabled) return;

    std::vector<Entity> ents;
    EnterCriticalSection(&g_cs); ents = g_entities; LeaveCriticalSection(&g_cs);

    int originX = g_winX + g_winW / 2, originY;
    int tp = cfg::tracer_pos.load();
    if (tp == 0) originY = g_winY + 2;
    else if (tp == 1) originY = g_winY + g_winH / 2;
    else originY = g_winY + g_winH - 2;

    float fovPx = GetAimFovPx();
    int drawn = 0, maxEsp = cfg::esp_max.load();

    for (auto& e : ents) {
        if (drawn >= maxEsp) break;
        bool locked = (!g_aimLock.empty() && e.name == g_aimLock);
        if (cfg::esp_lock_only && !locked) continue;

        COLORREF col = e.visible ? EspColorVisible() : EspColorHidden();
        if (cfg::esp_rainbow) col = Rainbow(rainT + e.dist * 0.01f);
        if (cfg::esp_hp_color && e.hp >= 0.f) col = HpColor(e.hp);
        if (cfg::esp_lock_hl && locked) col = RGB(255, 255, 0);

        if (cfg::esp_offscreen && e.valid && !e.visible)
            OffscreenArrow(hdc, e.screenHead.x, e.screenHead.y, cx, cy, col);

        if (!e.valid) continue;
        if (cfg::esp_fov_only) {
            float dx = e.screenHead.x - cx, dy = e.screenHead.y - cy;
            if (sqrtf(dx * dx + dy * dy) > fovPx) continue;
        }

        float hx = e.screenHead.x, hy = e.screenHead.y;
        if (hx < g_winX - 200 || hx > g_winX + g_winW + 200) continue;
        if (hy < g_winY - 200 || hy > g_winY + g_winH + 200) continue;

        int boxX = (int)(hx - e.boxW * 0.5f);
        int boxY = (int)(hy - e.boxH * 0.12f);
        int bw = (int)e.boxW, bh = (int)e.boxH;
        int lt = locked && cfg::esp_lock_hl ? t + 1 : t;

        if (cfg::esp_filled) FilledBox(hdc, boxX, boxY, bw, bh, col);
        if (cfg::esp_box) Box(hdc, boxX, boxY, bw, bh, col, lt);
        if (cfg::esp_outline) Box(hdc, boxX - 1, boxY - 1, bw + 2, bh + 2, RGB(0, 0, 0), 1);
        if (cfg::esp_corner) CornerBox(hdc, boxX, boxY, bw, bh, col, lt);
        bool drawSkel = cfg::esp_skeleton || (cfg::esp_skel_lock && locked);
        if (drawSkel) DrawSkeleton(hdc, e, col);
        if (cfg::esp_china_hat) ChinaHat(hdc, hx, hy, e.boxH, col);
        if (cfg::esp_healthbar) HealthBar(hdc, boxX, boxY, bh, e.hp);
        if (cfg::esp_tracer) {
            float tx = e.rootValid ? e.screenRoot.x : hx;
            float ty = e.rootValid ? e.screenRoot.y : hy;
            Line(hdc, originX, originY, (int)tx, (int)ty, col, lt);
        }
        if (cfg::esp_head)
            FillCircle(hdc, (int)hx, (int)hy, max(2, (int)(e.boxH * 0.06f)), RGB(255, 40, 40));
        if (cfg::esp_name) {
            SetBkMode(hdc, TRANSPARENT);
            SetTextColor(hdc, locked ? RGB(255, 255, 0) : RGB(255, 255, 255));
            char buf[96];
            if (cfg::stream_mode) sprintf_s(buf, "[%.0f]", e.dist);
            else if (e.hp >= 0.f) sprintf_s(buf, "%s [%.0f] hp:%.0f", e.name.c_str(), e.dist, e.hp);
            else sprintf_s(buf, "%s [%.0f]", e.name.c_str(), e.dist);
            TextOutA(hdc, (int)hx - 30, boxY - 16, buf, (int)strlen(buf));
        }
        drawn++;
    }
}

LRESULT CALLBACK OverlayProc(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp) {
    if (msg == WM_PAINT) {
        PAINTSTRUCT ps; HDC hdc = BeginPaint(hwnd, &ps);
        RECT rc; GetClientRect(hwnd, &rc);
        HDC mem = CreateCompatibleDC(hdc);
        HBITMAP bmp = CreateCompatibleBitmap(hdc, rc.right, rc.bottom);
        SelectObject(mem, bmp);
        HBRUSH br = CreateSolidBrush(RGB(1, 0, 1));
        FillRect(mem, &rc, br); DeleteObject(br);
        DrawEsp(mem); DrawRadar(mem);
        BitBlt(hdc, 0, 0, rc.right, rc.bottom, mem, 0, 0, SRCCOPY);
        DeleteObject(bmp); DeleteDC(mem); EndPaint(hwnd, &ps);
        return 0;
    }
    if (msg == WM_TIMER) { InvalidateRect(hwnd, nullptr, FALSE); return 0; }
    return DefWindowProc(hwnd, msg, wp, lp);
}

void CreateOverlay(HINSTANCE hi) {
    WNDCLASSA wc{};
    wc.lpfnWndProc = OverlayProc; wc.hInstance = hi; wc.lpszClassName = "meowya_ov";
    RegisterClassA(&wc);
    int sw = GetSystemMetrics(SM_CXSCREEN), sh = GetSystemMetrics(SM_CYSCREEN);
    g_overlay = CreateWindowExA(
        WS_EX_TOPMOST | WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_TOOLWINDOW,
        "meowya_ov", "", WS_POPUP, 0, 0, sw, sh, 0, 0, hi, 0);
    SetLayeredWindowAttributes(g_overlay, RGB(1, 0, 1), 0, LWA_COLORKEY);
    ShowWindow(g_overlay, SW_SHOW);
    SetTimer(g_overlay, 1, 3, nullptr);
}

void Chk(HWND p, const char* t, int id, int x, int y, bool on) {
    HWND h = CreateWindowA("BUTTON", t, WS_CHILD | WS_VISIBLE | BS_AUTOCHECKBOX,
        x, y, 175, 14, p, (HMENU)(INT_PTR)id, 0, 0);
    if (on) SendMessage(h, BM_SETCHECK, BST_CHECKED, 0);
}
void Section(HWND p, const char* t, int x, int y) {
    CreateWindowA("STATIC", t, WS_CHILD | WS_VISIBLE, x, y, 400, 13, p, 0, 0, 0);
}

LRESULT CALLBACK MenuProc(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp) {
    switch (msg) {
    case WM_CREATE: {
        HFONT font = CreateFontA(11, 0, 0, 0, FW_NORMAL, 0, 0, 0, ANSI_CHARSET,
            0, 0, CLEARTYPE_QUALITY, FIXED_PITCH | FF_MODERN, "Consolas");
        HFONT big = CreateFontA(17, 0, 0, 0, FW_BOLD, 0, 0, 0, ANSI_CHARSET,
            0, 0, CLEARTYPE_QUALITY, FIXED_PITCH | FF_MODERN, "Consolas");
        HWND title = CreateWindowA("STATIC", "  MEOWYA",
            WS_CHILD | WS_VISIBLE | SS_CENTER, 0, 2, 450, 16, hwnd, 0, 0, 0);
        SendMessage(title, WM_SETFONT, (WPARAM)big, 1);
        CreateWindowA("STATIC", "  F1 ESP  F2 Aim  F3 Radar  F4 Trig | INS HOME END",
            WS_CHILD | WS_VISIBLE | SS_CENTER, 0, 18, 450, 12, hwnd, 0, 0, 0);

        int y = 34;
        Section(hwnd, "  -- VISUALS --", 4, y); y += 13;
        Chk(hwnd, "ESP master", 10, 12, y, false); Chk(hwnd, "Rainbow", 49, 200, y, false); y += 13;
        Chk(hwnd, "Box", 11, 12, y, false); Chk(hwnd, "Corner", 47, 200, y, false); y += 13;
        Chk(hwnd, "Filled", 54, 12, y, false); Chk(hwnd, "Outline", 57, 200, y, false); y += 13;
        Chk(hwnd, "Skeleton", 42, 12, y, false); Chk(hwnd, "Skel lock only", 74, 200, y, false); y += 13;
        Chk(hwnd, "China hat", 40, 12, y, false); Chk(hwnd, "Head", 12, 200, y, false); y += 13;
        Chk(hwnd, "Name", 13, 12, y, false); Chk(hwnd, "Health bar", 48, 200, y, false); y += 13;
        Chk(hwnd, "Tracers", 27, 12, y, false); Chk(hwnd, "HP color", 75, 200, y, false); y += 13;
        Chk(hwnd, "FOV only", 55, 12, y, false); Chk(hwnd, "Offscreen", 58, 200, y, false); y += 13;
        Chk(hwnd, "Lock highlight", 76, 12, y, true); Chk(hwnd, "Lock ESP only", 110, 200, y, false); y += 13;
        Chk(hwnd, "Radar", 14, 12, y, false); Chk(hwnd, "Radar rings", 77, 200, y, false); y += 13;
        Chk(hwnd, "Watermark", 56, 12, y, true); Chk(hwnd, "Keylist", 78, 200, y, true); y += 13;
        Chk(hwnd, "Danger banner", 90, 12, y, true); Chk(hwnd, "Stream mode", 79, 200, y, false); y += 13;
        Chk(hwnd, "Bullet tracers", 111, 12, y, false); Chk(hwnd, "Lock info card", 112, 200, y, true); y += 13;
        Chk(hwnd, "Pred mark", 130, 12, y, true); Chk(hwnd, "FOV on aim only", 131, 200, y, false); y += 15;

        Section(hwnd, "  -- COMBAT --", 4, y); y += 13;
        Chk(hwnd, "Aimbot RMB", 18, 12, y, false); Chk(hwnd, "Silent LMB", 21, 200, y, false); y += 13;
        Chk(hwnd, "Sticky", 28, 12, y, true); Chk(hwnd, "Triggerbot", 62, 200, y, false); y += 13;
        Chk(hwnd, "Aim body", 63, 12, y, false); Chk(hwnd, "Multipoint", 80, 200, y, false); y += 13;
        Chk(hwnd, "Closest dist", 113, 12, y, false); Chk(hwnd, "Flick aim", 81, 200, y, false); y += 13;
        Chk(hwnd, "Vis only aim", 64, 12, y, false); Chk(hwnd, "Front only", 91, 200, y, false); y += 13;
        Chk(hwnd, "Aim on shot", 132, 12, y, false); Chk(hwnd, "Low HP prio", 59, 200, y, false); y += 13;
        Chk(hwnd, "Hitmarker", 92, 12, y, true); Chk(hwnd, "Hit sound", 103, 200, y, true); y += 13;
        Chk(hwnd, "Lock beep", 60, 12, y, false); Chk(hwnd, "Aim line", 52, 200, y, false); y += 13;
        Chk(hwnd, "Nearby alert", 82, 12, y, false); Chk(hwnd, "Wall check", 29, 200, y, false); y += 13;
        Chk(hwnd, "Team check", 20, 12, y, false); Chk(hwnd, "Skip dead", 19, 200, y, false); y += 13;
        Chk(hwnd, "Crosshair", 51, 12, y, false); Chk(hwnd, "Crosshair dot", 83, 200, y, false); y += 15;

        Section(hwnd, "  -- MISC --", 4, y); y += 13;
        Chk(hwnd, "Auto bhop", 46, 12, y, false); Chk(hwnd, "Auto sprint", 65, 200, y, false); y += 13;
        Chk(hwnd, "Anti AFK", 84, 12, y, false); Chk(hwnd, "Show velocity", 66, 200, y, false); y += 13;
        Chk(hwnd, "Hide console", 67, 12, y, false); Chk(hwnd, "Menu topmost", 133, 200, y, true); y += 13;
        Chk(hwnd, "Cam W2S", 41, 12, y, true); Chk(hwnd, "Flip Y", 15, 200, y, false); y += 13;
        Chk(hwnd, "Matrix 330", 16, 12, y, false); Chk(hwnd, "Formula 2", 17, 200, y, false); y += 15;

        Section(hwnd, "  -- VALUES --", 4, y); y += 13;
        CreateWindowA("STATIC", "FOV%", WS_CHILD | WS_VISIBLE, 12, y, 28, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "15", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 42, y - 2, 28, 14, hwnd, (HMENU)33, 0, 0);
        CreateWindowA("STATIC", "Sm", WS_CHILD | WS_VISIBLE, 76, y, 16, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "1", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 94, y - 2, 24, 14, hwnd, (HMENU)34, 0, 0);
        CreateWindowA("STATIC", "Lead", WS_CHILD | WS_VISIBLE, 124, y, 28, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "0", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 152, y - 2, 24, 14, hwnd, (HMENU)35, 0, 0);
        CreateWindowA("STATIC", "MaxD", WS_CHILD | WS_VISIBLE, 182, y, 28, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "2000", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 212, y - 2, 36, 14, hwnd, (HMENU)53, 0, 0);
        CreateWindowA("STATIC", "AimD", WS_CHILD | WS_VISIBLE, 254, y, 28, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "2000", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 284, y - 2, 36, 14, hwnd, (HMENU)85, 0, 0);
        CreateWindowA("STATIC", "Thk", WS_CHILD | WS_VISIBLE, 326, y, 22, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "1", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 350, y - 2, 24, 14, hwnd, (HMENU)61, 0, 0);
        y += 16;
        CreateWindowA("STATIC", "TrigMs", WS_CHILD | WS_VISIBLE, 12, y, 40, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "40", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 54, y - 2, 32, 14, hwnd, (HMENU)68, 0, 0);
        CreateWindowA("STATIC", "Near", WS_CHILD | WS_VISIBLE, 96, y, 28, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "50", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 126, y - 2, 32, 14, hwnd, (HMENU)86, 0, 0);
        CreateWindowA("STATIC", "EspMax", WS_CHILD | WS_VISIBLE, 168, y, 40, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "32", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 210, y - 2, 32, 14, hwnd, (HMENU)134, 0, 0);
        y += 16;
        CreateWindowA("STATIC", "Vis", WS_CHILD | WS_VISIBLE, 12, y, 22, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "0", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 36, y - 2, 26, 14, hwnd, (HMENU)30, 0, 0);
        CreateWindowA("EDIT", "255", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 66, y - 2, 26, 14, hwnd, (HMENU)31, 0, 0);
        CreateWindowA("EDIT", "100", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 96, y - 2, 26, 14, hwnd, (HMENU)32, 0, 0);
        CreateWindowA("STATIC", "Hid", WS_CHILD | WS_VISIBLE, 132, y, 22, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "255", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 156, y - 2, 26, 14, hwnd, (HMENU)43, 0, 0);
        CreateWindowA("EDIT", "60", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 186, y - 2, 26, 14, hwnd, (HMENU)44, 0, 0);
        CreateWindowA("EDIT", "60", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 216, y - 2, 26, 14, hwnd, (HMENU)45, 0, 0);
        y += 16;
        CreateWindowA("STATIC", "Trace", WS_CHILD | WS_VISIBLE, 12, y, 36, 11, hwnd, 0, 0, 0);
        CreateWindowA("EDIT", "255", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 52, y - 2, 28, 14, hwnd, (HMENU)120, 0, 0);
        CreateWindowA("EDIT", "200", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 84, y - 2, 28, 14, hwnd, (HMENU)121, 0, 0);
        CreateWindowA("EDIT", "50", WS_CHILD | WS_VISIBLE | WS_BORDER | ES_NUMBER, 116, y - 2, 28, 14, hwnd, (HMENU)122, 0, 0);
        y += 18;
        CreateWindowA("STATIC", "status", WS_CHILD | WS_VISIBLE, 12, y, 410, 32, hwnd, (HMENU)50, 0, 0);

        EnumChildWindows(hwnd, [](HWND c, LPARAM f) -> BOOL {
            SendMessage(c, WM_SETFONT, f, 1); return TRUE;
            }, (LPARAM)font);
        SendMessage(title, WM_SETFONT, (WPARAM)big, 1);
        SetTimer(hwnd, 1, 100, 0);
        break;
    }
    case WM_COMMAND: {
        if (HIWORD(wp) != BN_CLICKED) break;
        int id = LOWORD(wp);
        bool on = SendMessage((HWND)lp, BM_GETCHECK, 0, 0) == BST_CHECKED;
        if (id == 10) cfg::esp_enabled = on;
        if (id == 11) cfg::esp_box = on;
        if (id == 12) cfg::esp_head = on;
        if (id == 13) cfg::esp_name = on;
        if (id == 14) cfg::radar_enabled = on;
        if (id == 15) cfg::flip_y = on;
        if (id == 16) cfg::use_mat_330 = on;
        if (id == 17) cfg::use_formula2 = on;
        if (id == 18) cfg::aimbot = on;
        if (id == 19) cfg::skip_dead = on;
        if (id == 20) cfg::team_check = on;
        if (id == 21) cfg::silent_aim = on;
        if (id == 27) cfg::esp_tracer = on;
        if (id == 28) cfg::aim_sticky = on;
        if (id == 29) cfg::wall_check = on;
        if (id == 40) cfg::esp_china_hat = on;
        if (id == 41) cfg::use_cam_w2s = on;
        if (id == 42) cfg::esp_skeleton = on;
        if (id == 46) cfg::auto_bhop = on;
        if (id == 47) cfg::esp_corner = on;
        if (id == 48) cfg::esp_healthbar = on;
        if (id == 49) cfg::esp_rainbow = on;
        if (id == 51) cfg::crosshair = on;
        if (id == 52) cfg::aim_line = on;
        if (id == 54) cfg::esp_filled = on;
        if (id == 55) cfg::esp_fov_only = on;
        if (id == 56) cfg::watermark = on;
        if (id == 57) cfg::esp_outline = on;
        if (id == 58) cfg::esp_offscreen = on;
        if (id == 59) cfg::aim_lowhp = on;
        if (id == 60) cfg::aim_beep = on;
        if (id == 62) cfg::triggerbot = on;
        if (id == 63) cfg::aim_body = on;
        if (id == 64) cfg::aim_vis_only = on;
        if (id == 65) cfg::auto_sprint = on;
        if (id == 66) cfg::show_vel = on;
        if (id == 67) cfg::hide_console = on;
        if (id == 74) cfg::esp_skel_lock = on;
        if (id == 75) cfg::esp_hp_color = on;
        if (id == 76) cfg::esp_lock_hl = on;
        if (id == 77) cfg::radar_rings = on;
        if (id == 78) cfg::keylist = on;
        if (id == 79) cfg::stream_mode = on;
        if (id == 80) cfg::aim_multipoint = on;
        if (id == 81) cfg::aim_flick = on;
        if (id == 82) cfg::nearby_alert = on;
        if (id == 83) cfg::crosshair_dot = on;
        if (id == 84) cfg::anti_afk = on;
        if (id == 90) cfg::danger_banner = on;
        if (id == 91) cfg::aim_front = on;
        if (id == 92) cfg::hitmarker = on;
        if (id == 103) cfg::hit_sound = on;
        if (id == 110) cfg::esp_lock_only = on;
        if (id == 111) cfg::bullet_tracers = on;
        if (id == 112) cfg::lock_info = on;
        if (id == 113) cfg::aim_closest = on;
        if (id == 130) cfg::pred_mark = on;
        if (id == 131) cfg::fov_on_aim = on;
        if (id == 132) cfg::aim_on_shot = on;
        if (id == 133) {
            cfg::menu_topmost = on;
            if (g_menu)
                SetWindowPos(g_menu, on ? HWND_TOPMOST : HWND_NOTOPMOST, 0, 0, 0, 0, SWP_NOMOVE | SWP_NOSIZE);
        }
        break;
    }
    case WM_TIMER: {
        char b[16];
        GetWindowTextA(GetDlgItem(hwnd, 30), b, 16); cfg::col_r = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 31), b, 16); cfg::col_g = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 32), b, 16); cfg::col_b = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 43), b, 16); cfg::col_hr = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 44), b, 16); cfg::col_hg = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 45), b, 16); cfg::col_hb = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 120), b, 16); cfg::bt_r = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 121), b, 16); cfg::bt_g = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 122), b, 16); cfg::bt_b = max(0, min(255, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 33), b, 16); cfg::aim_fov_pct = (float)max(3, min(50, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 34), b, 16); cfg::aim_smooth = (float)max(1, min(20, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 35), b, 16); cfg::aim_lead = atoi(b) / 100.f;
        GetWindowTextA(GetDlgItem(hwnd, 53), b, 16); cfg::max_dist = max(50, min(10000, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 85), b, 16); cfg::aim_max_dist = max(50, min(10000, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 61), b, 16); cfg::line_thick = max(1, min(4, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 68), b, 16); cfg::trigger_delay = max(10, min(500, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 86), b, 16); cfg::nearby_studs = max(10, min(500, atoi(b)));
        GetWindowTextA(GetDlgItem(hwnd, 134), b, 16); cfg::esp_max = max(4, min(64, atoi(b)));
        char buf[240];
        sprintf_s(buf, "local: %s\n%s", g_localName.c_str(), g_status.c_str());
        SetWindowTextA(GetDlgItem(hwnd, 50), buf);
        break;
    }
    case WM_CTLCOLORSTATIC:
    case WM_CTLCOLORBTN: {
        HDC hdc = (HDC)wp;
        SetTextColor(hdc, RGB(220, 230, 240));
        SetBkColor(hdc, RGB(14, 14, 20));
        static HBRUSH br = CreateSolidBrush(RGB(14, 14, 20));
        return (LRESULT)br;
    }
    case WM_ERASEBKGND: {
        RECT rc; GetClientRect(hwnd, &rc);
        HBRUSH br = CreateSolidBrush(RGB(14, 14, 20));
        FillRect((HDC)wp, &rc, br); DeleteObject(br);
        return 1;
    }
    case WM_DESTROY:
        g_running = false; PostQuitMessage(0); break;
    }
    return DefWindowProc(hwnd, msg, wp, lp);
}

int WINAPI WinMain(HINSTANCE hi, HINSTANCE, LPSTR, int) {
    InitializeCriticalSection(&g_cs);
    AllocConsole();
    freopen_s((FILE**)stdout, "CONOUT$", "w", stdout);
    SetConsoleTitleA("meowya");
    system("chcp 65001 > nul");

    system("color 04");


    printf("\n                   ███╗   ███╗ ███████╗  ██████╗  ██╗    ██╗ ██╗   ██╗ █████╗");
    system("color 04");


    printf("\n                   ████╗ ████║ ██╔════╝ ██╔═══██╗ ██║    ██║ ╚██╗ ██╔╝██╔══██╗");
    system("color 04");


    printf("\n                   ██╔████╔██║ █████╗   ██║   ██║ ██║ █╗ ██║  ╚████╔╝ ███████║");
    system("color 04");


    printf("\n                   ██║╚██╔╝██║ ██╔══╝   ██║   ██║ ██║███╗██║   ╚██╔╝  ██╔══██║");
    system("color 04");


    printf("\n                   ██║ ╚═╝ ██║ ███████╗ ╚██████╔╝ ╚███╔███╔╝    ██║   ██║  ██║");
    system("color 04");


    printf("\n                   ╚═╝     ╚═╝ ╚══════╝  ╚═════╝   ╚══╝╚══╝     ╚═╝   ╚═╝  ╚═╝");
    system("color 04");


    printf("\n                                    discord.gg/9vnWNDasVz\n\n");
    printf("\n                   welcome to meowya we thank you for using meowya have fun!\n\n");
    system("color 04");
    printf("\n                      ██");

    system("color 04");
        printf("\n                   ████            ██████████████");

    system("color 04");
    printf("\n    ██            ████      ████████████████████████");

    system("color 04");
    printf("\n    ████    ████████████  ██████████████████████████████");

    system("color 04");
    printf("\n    ██████████████████████████████████████████████████████");

    system("color 04");
    printf("\n    ██████   ██   ███████████████████████████████████████");

    system("color 04");
    printf("\n    ██████   ██   █████████████████████████████████████ ");

    system("color 04");
    printf("\n    ██████████████████████████████████████████████████████");

    system("color 04");
    printf("\n    █████████    ████████  ██████████████████████████████");

    system("color 04");
    printf("\n    ████████████████  ████████████████████████████████");

    system("color 04");
    printf("\n    ███████████████  ██████████████████████████████████");

    system("color 04");
    printf("\n    ██████████████  ██████████████████████████████████");

    system("color 04");
    printf("\n    ████████████  ██████████████████████████████████");

    system("color 04");
    printf("\n    ████████████  ████████████████████████████████");

    system("color 04");
    printf("\n    ████████████  ████████████████████████████████");

    system("color 04");
    printf("\n    ██████████    ████████████████████████████████");

    system("color 04");
    printf("\n    ███████       ████████████████    ██████████");


    system("color 04");
    printf("\n                                    ██████████  ");

    system("color 04");
    printf("\n                                   ██████████");

    system("color 04");
    printf("\n                                  ██████████");

    system("color 04");
    printf("\n                         ██████████████████");

    system("color 04");
    printf("\n                        ██████████████████");

    printf("\n              DO NOT CLOSE THIS IT WILL CLOSE THE CHEAT!");
    printf("Pred mark | FOV on aim | Aim on shot | Menu topmost | EspMax\n");
    printf("RUN AS ADMIN\n\n");

    CreateOverlay(hi);
    WNDCLASSA wc{};
    wc.lpfnWndProc = MenuProc; wc.hInstance = hi;
    wc.lpszClassName = "meowya_m"; wc.hCursor = LoadCursor(0, IDC_ARROW);
    RegisterClassA(&wc);
    g_menu = CreateWindowExA(WS_EX_TOPMOST, "meowya_m", "meowya",
        WS_OVERLAPPED | WS_CAPTION | WS_SYSMENU | WS_MINIMIZEBOX,
        40, 10, 460, 860, 0, 0, hi, 0);
    ShowWindow(g_menu, SW_SHOW);
    std::thread(Worker).detach();

    MSG msg;
    while (GetMessage(&msg, 0, 0, 0)) {
        TranslateMessage(&msg); DispatchMessage(&msg);
    }
    g_running = false;
    DeleteCriticalSection(&g_cs);
    return 0;
}
