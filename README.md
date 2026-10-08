-- ================================================================= --
--   DARK SHADOW HUB v4.2  |  LUXURY EDITION                          --
--   Merge a Mini Army: Auto Merge ANDANDO (sem teleporte),           --
--   Auto Rebirth / Upgrade silenciosos, efeitos sonoros com volume,  --
--   scripts por jogo (só aparecem no jogo certo), Anti-Lag e Mobile. --
-- ================================================================= --
--  ÍNDICE
--   1. Serviços e ambiente
--   2. Utilitários, tema e sons
--   3. Estado global
--   4. Interface base (janela, sidebar, header)
--   5. Framework de abas e sub-abas
--   6. Notificações
--   7. Componentes
--   8. Busca
--   9. Núcleo: movimento, visuais, Performance Guard, ESP, hook
--  10. Abas: Home / Player / Main / Settings
--  11. Sistema de scripts por jogo
--  12. Jogo: Merge a Mini Army
--  13. Loops, abrir/fechar, arraste, unload, inicialização
-- ================================================================= --

if not game:IsLoaded() then game.Loaded:Wait() end

-- ==========================================
-- 1. SERVIÇOS E AMBIENTE
-- ==========================================
local TweenService     = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService       = game:GetService("RunService")
local Players          = game:GetService("Players")
local Lighting         = game:GetService("Lighting")
local TeleportService  = game:GetService("TeleportService")
local GuiService       = game:GetService("GuiService")

local LocalPlayer = Players.LocalPlayer
local ParentGui   = (gethui and gethui()) or LocalPlayer:WaitForChild("PlayerGui")
local env         = (getgenv and getgenv()) or _G

-- Descarrega instância anterior (evita conexões duplicadas)
if env.__ShadowHubUnload then pcall(env.__ShadowHubUnload) end
for _, n in ipairs({"ShadowTechHub", "ShadowESP"}) do
	local o = ParentGui:FindFirstChild(n)
	if o then o:Destroy() end
end

-- ==========================================
-- 2. UTILITÁRIOS, TEMA E SONS
-- ==========================================
local IS_TOUCH = UserInputService.TouchEnabled -- celular / emulador

local connections = {}
local function track(conn)
	table.insert(connections, conn)
	return conn
end

local function create(class, props)
	local inst = Instance.new(class)
	local parent
	for k, v in pairs(props or {}) do
		if k == "Parent" then parent = v else inst[k] = v end
	end
	inst.Parent = parent
	return inst
end

local function corner(inst, r)
	return create("UICorner", {CornerRadius = UDim.new(0, r), Parent = inst})
end

local COLORS = {
	Background   = Color3.fromRGB(15, 10, 18),
	Sidebar      = Color3.fromRGB(22, 14, 26),
	Card         = Color3.fromRGB(26, 17, 30),
	CardHeader   = Color3.fromRGB(36, 23, 42),
	CardHover    = Color3.fromRGB(46, 29, 54),
	SubTabBg     = Color3.fromRGB(30, 18, 35),
	SubTabActive = Color3.fromRGB(48, 28, 56),
	Accent       = Color3.fromRGB(230, 75, 150),
	AccentDark   = Color3.fromRGB(180, 50, 110),
	AccentGlow   = Color3.fromRGB(255, 110, 180),
	TextMain     = Color3.fromRGB(245, 240, 250),
	TextDark     = Color3.fromRGB(140, 125, 150),
	Stroke       = Color3.fromRGB(48, 30, 56),
	SwitchOff    = Color3.fromRGB(45, 30, 52),
	Green        = Color3.fromRGB(80, 220, 120),
	Danger       = Color3.fromRGB(200, 60, 80),
	Gold         = Color3.fromRGB(255, 205, 60),
}

local function stroke(inst, color, thickness, transparency)
	return create("UIStroke", {
		Color = color or COLORS.Stroke,
		Thickness = thickness or 1,
		Transparency = transparency or 0,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		Parent = inst,
	})
end

local function tween(inst, t, props, style, dir)
	local tw = TweenService:Create(inst, TweenInfo.new(t, style or Enum.EasingStyle.Quart, dir or Enum.EasingDirection.Out), props)
	tw:Play()
	return tw
end

local function label(props)
	props.BackgroundTransparency = 1
	props.Font = props.Font or Enum.Font.GothamMedium
	props.TextColor3 = props.TextColor3 or COLORS.TextMain
	props.TextSize = props.TextSize or 12
	props.TextXAlignment = props.TextXAlignment or Enum.TextXAlignment.Left
	return create("TextLabel", props)
end

local orderCounters = setmetatable({}, {__mode = "k"})
local function nextOrder(obj)
	orderCounters[obj] = (orderCounters[obj] or 0) + 1
	return orderCounters[obj]
end

local function hasAny(text, words)
	for i = 1, #words do
		if string.find(text, words[i], 1, true) then return true end
	end
	return false
end

local function splitWords(text)
	local words = {}
	for w in string.gmatch(string.lower(text or ""), "[^,;]+") do
		w = string.gsub(w, "^%s*(.-)%s*$", "%1")
		if #w > 0 then words[#words + 1] = w end
	end
	return words
end

-- Cache do personagem (evita FindFirstChild a cada frame)
local cHum, cRoot
local function getHum()
	local c = LocalPlayer.Character
	if not c then return nil end
	if cHum and cHum.Parent == c then return cHum end
	cHum = c:FindFirstChildOfClass("Humanoid")
	return cHum
end

local function getRoot()
	local c = LocalPlayer.Character
	if not c then return nil end
	if cRoot and cRoot.Parent == c then return cRoot end
	cRoot = c:FindFirstChild("HumanoidRootPart")
	return cRoot
end

local clipboardFn = setclipboard or toclipboard or nil

-- ICONES (troque os IDs se algum não carregar)
local ICONS = {
	Logo     = "rbxassetid://10723415903",
	Home     = "rbxassetid://10723354417",
	Main     = "rbxassetid://10734950309",
	Player   = "rbxassetid://10747373176",
	Settings = "rbxassetid://10734950020",
	Search   = "rbxassetid://10734938520",
	Merge    = "rbxassetid://10734976722",
	Economy  = "rbxassetid://10734937986",
	Combat   = "rbxassetid://10734938210",
	Server   = "rbxassetid://10734938384",
	Shield   = "rbxassetid://10734938450",
	Bolt     = "rbxassetid://10734950309",
}

-- EFEITOS SONOROS
-- Um único som base com tons diferentes (grave = desligar/erro, agudo = ligar/aviso).
-- Se quiser outro som, troque só o Id abaixo. O volume é ajustado em Settings > Sound.
local Sfx = {
	Enabled = true,
	Volume  = 0.5,
	Id      = "rbxassetid://6895079853",
	pool    = {},
	last    = {},
	holder  = nil, -- definido depois que a ScreenGui existe
}
local SFX_KINDS = {            -- {tom, ganho}
	Click  = {1.00, 1.0},
	On     = {1.30, 1.0},
	Off    = {0.80, 1.0},
	Open   = {1.10, 1.0},
	Close  = {0.85, 1.0},
	Notify = {1.60, 0.8},
	Error  = {0.60, 1.0},
	Slide  = {1.25, 0.4},
}

function Sfx.Play(kind, minGap)
	if not Sfx.Enabled or Sfx.Volume <= 0 or not Sfx.holder then return end
	local now = os.clock()
	if minGap and Sfx.last[kind] and now - Sfx.last[kind] < minGap then return end
	Sfx.last[kind] = now
	local k = SFX_KINDS[kind] or SFX_KINDS.Click
	local s = Sfx.pool[kind]
	if not s or not s.Parent then
		s = Instance.new("Sound")
		s.Name = "Sfx_" .. kind
		s.SoundId = Sfx.Id
		s.Parent = Sfx.holder
		Sfx.pool[kind] = s
	end
	pcall(function()
		s.PlaybackSpeed = k[1]
		s.Volume = math.clamp(Sfx.Volume * k[2], 0, 1)
		s.TimePosition = 0
		s:Play()
	end)
end

-- ==========================================
-- 3. ESTADO GLOBAL
-- ==========================================
local isOpen = false
local userScale = 1
local currentScale = 1
local toggleKey = Enum.KeyCode.RightShift
local listeningKey = false
local notificationsEnabled = true
local activeTab = "Home"
local startTime = os.time()
local origMaxZoom = LocalPlayer.CameraMaxZoomDistance

local State = {
	Speed = 50, SpeedOn = false,
	Jump = 100, JumpOn = false,
	InfJump = false, Noclip = false,
	Fly = false, FlySpeed = 60,
	Fullbright = false, NoFog = false,
	FovOn = false, Fov = 70,
	AntiAfk = false, LowGfx = false,
	NoRender = false, AutoRejoin = false,
	Snow = not IS_TOUCH,
}
local Original = {}

-- Movimento (agrupado numa tabela para economizar variáveis locais)
local Mv = {touched = {}, parts = {}, refresh = 0, fly = nil, vel = Vector3.zero, controls = nil}
pcall(function()
	Mv.controls = require(LocalPlayer:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 3)):GetControls()
end)

-- Performance Guard (anti-lag automático)
local Perf = {
	Guard = true, LowFps = 35, CritFps = 22, AutoRestore = true,
	Fps = 60, lowSec = 0, critSec = 0, goodSec = 0, flaps = 0,
	autoAL = false, autoFB = false, alOn = false, ultraOn = false,
	Lite = false, Status = "Monitoring", gfx = nil,
}
local PerfUI = {}  -- referências dos toggles de performance
local Refs = {}    -- referências de labels da interface

-- ==========================================
-- 4. INTERFACE BASE
-- ==========================================
local ScreenGui = create("ScreenGui", {
	Name = "ShadowTechHub", ResetOnSpawn = false, IgnoreGuiInset = true,
	DisplayOrder = 999, ZIndexBehavior = Enum.ZIndexBehavior.Sibling, Parent = ParentGui,
})
Sfx.holder = ScreenGui

local InputBlocker = create("TextButton", {
	Name = "InputBlocker", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
	Text = "", Visible = false, Modal = true, ZIndex = 1, Parent = ScreenGui,
})

-- Neve (leve, só com o menu aberto e sem Lite)
local SnowContainer = create("Frame", {
	Name = "SnowContainer", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
	ClipsDescendants = true, Visible = false, ZIndex = 1, Parent = ScreenGui,
})
local updateSnow
do
	local flakes = {}
	for i = 1, (IS_TOUCH and 14 or 26) do
		local size = math.random(3, 7)
		local frame = create("Frame", {
			Size = UDim2.fromOffset(size, size), BackgroundColor3 = Color3.fromRGB(255, 230, 245),
			BackgroundTransparency = math.random(15, 50) / 100, BorderSizePixel = 0, Parent = SnowContainer,
		})
		corner(frame, 8)
		flakes[i] = {frame = frame, x = math.random(), y = math.random(),
			speed = math.random(80, 220) / 1000, drift = math.random(10, 40) / 10, seed = math.random(1, 100)}
	end
	updateSnow = function(dt)
		local now = os.clock()
		for i = 1, #flakes do
			local f = flakes[i]
			f.y = f.y + f.speed * dt
			f.x = (f.x + math.sin(now * f.drift + f.seed) * 0.015 * dt) % 1
			if f.y > 1.05 then f.y = -0.05; f.x = math.random() end
			f.frame.Position = UDim2.fromScale(f.x, f.y)
		end
	end
end

-- Botão flutuante (pulsa de leve para chamar atenção)
local FloatingBtn = create("Frame", {
	Name = "FloatingButton", Size = UDim2.fromOffset(48, 48), Position = UDim2.new(0, 16, 0.3, 0),
	BackgroundColor3 = COLORS.Sidebar, BorderSizePixel = 0, Active = true, ZIndex = 10, Parent = ScreenGui,
})
corner(FloatingBtn, 14)
local BtnStroke = stroke(FloatingBtn, COLORS.Accent, 1.5, 0.3)
create("ImageLabel", {
	Size = UDim2.fromOffset(24, 24), AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
	BackgroundTransparency = 1, Image = ICONS.Logo, ImageColor3 = COLORS.Accent, Parent = FloatingBtn,
})
pcall(function()
	TweenService:Create(BtnStroke, TweenInfo.new(1.3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true),
		{Thickness = 3}):Play()
end)

-- Janela principal (tamanho menor no celular)
local HUB_W, HUB_H, SIDEBAR_W = 660, 420, 150
if IS_TOUCH then HUB_W, HUB_H, SIDEBAR_W = 580, 340, 132 end
local hubCenter = Vector2.new(0, 0)

local MainFrame = create("CanvasGroup", {
	Name = "MainHub", Size = UDim2.fromOffset(HUB_W, HUB_H), AnchorPoint = Vector2.new(0.5, 0.5),
	BackgroundColor3 = COLORS.Background, BorderSizePixel = 0, Visible = false, Active = true,
	ZIndex = 5, GroupTransparency = 1, Parent = ScreenGui,
})
corner(MainFrame, 16)
stroke(MainFrame, COLORS.Stroke, 1.2)
local HubScale = create("UIScale", {Scale = 1, Parent = MainFrame})

local DragGhost = create("Frame", {
	Name = "DragGhost", AnchorPoint = Vector2.new(0.5, 0.5), Visible = false, BackgroundTransparency = 0.85,
	BackgroundColor3 = COLORS.Accent, BorderSizePixel = 0, ZIndex = 20, Parent = ScreenGui,
})
corner(DragGhost, 16)
stroke(DragGhost, COLORS.AccentGlow, 1.8)

local function screenSize() return ScreenGui.AbsoluteSize end
local function hubPixelSize() return Vector2.new(HUB_W, HUB_H) * currentScale end

local function clampCenter(c)
	local s, half = screenSize(), hubPixelSize() / 2
	local x = math.clamp(c.X, half.X, math.max(half.X, s.X - half.X))
	local y = math.clamp(c.Y, half.Y, math.max(half.Y, s.Y - half.Y))
	return Vector2.new(x, y)
end

local function computeScale()
	local s = screenSize()
	local fit = math.min(1, (s.X - 24) / HUB_W, (s.Y - 24) / HUB_H)
	return math.max(0.4, fit) * userScale
end

-- Sidebar
local Sidebar, TabsContainer, AvatarImg
do
	Sidebar = create("Frame", {
		Size = UDim2.new(0, SIDEBAR_W, 1, 0), BackgroundColor3 = COLORS.Sidebar,
		BorderSizePixel = 0, Active = true, Parent = MainFrame,
	})
	create("Frame", {
		AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromScale(1, 0), Size = UDim2.new(0, 1, 1, 0),
		BackgroundColor3 = COLORS.Stroke, BorderSizePixel = 0, Parent = Sidebar,
	})
	local logo = create("Frame", {Size = UDim2.new(1, 0, 0, 56), BackgroundTransparency = 1, Parent = Sidebar})
	create("ImageLabel", {
		Size = UDim2.fromOffset(26, 26), Position = UDim2.new(0, 12, 0.5, -13),
		BackgroundTransparency = 1, Image = ICONS.Logo, ImageColor3 = COLORS.Accent, Parent = logo,
	})
	local title = label({Size = UDim2.new(1, -46, 0, 16), Position = UDim2.new(0, 44, 0.5, -15), Text = "DARK SHADOW",
		Font = Enum.Font.GothamBlack, TextSize = 11, TextTruncate = Enum.TextTruncate.AtEnd, Parent = logo})
	create("UIGradient", {
		Color = ColorSequence.new(COLORS.TextMain, COLORS.AccentGlow), Parent = title,
	})
	label({Size = UDim2.new(1, -46, 0, 12), Position = UDim2.new(0, 44, 0.5, 2), Text = "HUB  v4.2",
		TextSize = 10, TextColor3 = COLORS.Accent, Parent = logo})

	TabsContainer = create("ScrollingFrame", {
		Size = UDim2.new(1, -16, 1, -128), Position = UDim2.new(0, 8, 0, 60),
		BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 0,
		AutomaticCanvasSize = Enum.AutomaticSize.Y, CanvasSize = UDim2.new(),
		ScrollingDirection = Enum.ScrollingDirection.Y, Parent = Sidebar,
	})
	create("UIListLayout", {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 6), Parent = TabsContainer})

	local profile = create("Frame", {
		Size = UDim2.new(1, -16, 0, 48), Position = UDim2.new(0, 8, 1, -54), BackgroundTransparency = 1, Parent = Sidebar,
	})
	local avFrame = create("Frame", {
		Size = UDim2.fromOffset(34, 34), Position = UDim2.new(0, 2, 0.5, -17), BackgroundTransparency = 1, Parent = profile,
	})
	AvatarImg = create("ImageLabel", {Size = UDim2.fromScale(1, 1), BackgroundColor3 = COLORS.Card, Parent = avFrame})
	corner(AvatarImg, 17)
	local dot = create("Frame", {
		Size = UDim2.fromOffset(10, 10), Position = UDim2.new(1, -8, 1, -8),
		BackgroundColor3 = COLORS.Green, BorderSizePixel = 0, ZIndex = 3, Parent = avFrame,
	})
	corner(dot, 5)
	stroke(dot, COLORS.Sidebar, 1.5)
	label({Size = UDim2.new(1, -44, 0, 16), Position = UDim2.new(0, 42, 0, 8), Text = LocalPlayer.DisplayName,
		Font = Enum.Font.GothamBold, TextTruncate = Enum.TextTruncate.AtEnd, Parent = profile})
	label({Size = UDim2.new(1, -44, 0, 14), Position = UDim2.new(0, 42, 0, 24), Text = "@" .. LocalPlayer.Name,
		TextSize = 10, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = profile})
end

-- Header + área de páginas
local TopHeader, HeaderIcon, HeaderTitle, MinimizeBtn, SearchBox, PagesContainer, NoResults
do
	local content = create("Frame", {
		Size = UDim2.new(1, -SIDEBAR_W, 1, 0), Position = UDim2.new(0, SIDEBAR_W, 0, 0),
		BackgroundTransparency = 1, Parent = MainFrame,
	})
	TopHeader = create("Frame", {
		Size = UDim2.new(1, -20, 0, 52), Position = UDim2.new(0, 10, 0, 4),
		BackgroundTransparency = 1, Active = true, Parent = content,
	})
	HeaderIcon = create("ImageLabel", {
		Size = UDim2.fromOffset(20, 20), Position = UDim2.new(0, 4, 0.5, -10),
		BackgroundTransparency = 1, ImageColor3 = COLORS.Accent, Image = ICONS.Home, Parent = TopHeader,
	})
	HeaderTitle = label({
		Size = UDim2.new(0, 110, 1, 0), Position = UDim2.new(0, 30, 0, 0), Text = "Home",
		Font = Enum.Font.GothamBold, TextSize = 15, TextTruncate = Enum.TextTruncate.AtEnd, Parent = TopHeader,
	})
	MinimizeBtn = create("TextButton", {
		Size = UDim2.fromOffset(30, 30), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, 0, 0.5, 0),
		BackgroundColor3 = COLORS.Card, Text = "—", TextColor3 = COLORS.TextDark, TextSize = 13,
		Font = Enum.Font.GothamBold, AutoButtonColor = false, Parent = TopHeader,
	})
	corner(MinimizeBtn, 8)
	MinimizeBtn.MouseEnter:Connect(function() tween(MinimizeBtn, 0.15, {BackgroundColor3 = COLORS.Danger, TextColor3 = COLORS.TextMain}) end)
	MinimizeBtn.MouseLeave:Connect(function() tween(MinimizeBtn, 0.15, {BackgroundColor3 = COLORS.Card, TextColor3 = COLORS.TextDark}) end)

	local sf = create("Frame", {
		Size = UDim2.fromOffset(IS_TOUCH and 130 or 170, 30), AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -38, 0.5, 0), BackgroundColor3 = COLORS.Card, Parent = TopHeader,
	})
	corner(sf, 8)
	local ss = stroke(sf, COLORS.Stroke, 1)
	create("ImageLabel", {
		Size = UDim2.fromOffset(14, 14), Position = UDim2.new(0, 9, 0.5, -7),
		BackgroundTransparency = 1, Image = ICONS.Search, ImageColor3 = COLORS.TextDark, Parent = sf,
	})
	SearchBox = create("TextBox", {
		Size = UDim2.new(1, -32, 1, 0), Position = UDim2.new(0, 28, 0, 0), BackgroundTransparency = 1,
		Text = "", PlaceholderText = "Search...", PlaceholderColor3 = COLORS.TextDark, TextColor3 = COLORS.TextMain,
		TextSize = 11, Font = Enum.Font.GothamMedium, TextXAlignment = Enum.TextXAlignment.Left,
		ClearTextOnFocus = false, Parent = sf,
	})
	SearchBox.Focused:Connect(function() tween(ss, 0.2, {Color = COLORS.Accent}) end)
	SearchBox.FocusLost:Connect(function() tween(ss, 0.2, {Color = COLORS.Stroke}) end)

	-- Linha de destaque com degradê abaixo do cabeçalho
	local accentLine = create("Frame", {
		Size = UDim2.new(1, -20, 0, 2), Position = UDim2.new(0, 10, 0, 56),
		BackgroundColor3 = COLORS.Accent, BorderSizePixel = 0, Parent = content,
	})
	create("UIGradient", {
		Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, COLORS.AccentDark),
			ColorSequenceKeypoint.new(0.5, COLORS.AccentGlow),
			ColorSequenceKeypoint.new(1, COLORS.AccentDark),
		}),
		Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.9),
			NumberSequenceKeypoint.new(0.5, 0.2),
			NumberSequenceKeypoint.new(1, 0.9),
		}),
		Parent = accentLine,
	})

	PagesContainer = create("Frame", {
		Size = UDim2.new(1, -20, 1, -68), Position = UDim2.new(0, 10, 0, 62), BackgroundTransparency = 1, Parent = content,
	})
	NoResults = label({
		Size = UDim2.fromScale(1, 1), Text = "No results found", TextSize = 13, TextColor3 = COLORS.TextDark,
		TextXAlignment = Enum.TextXAlignment.Center, Visible = false, ZIndex = 0, Parent = PagesContainer,
	})
end

-- ==========================================
-- 5. FRAMEWORK DE ABAS E SUB-ABAS
-- ==========================================
local tabs = {}
local groups = {}
local ORDER = {Home = 10, Player = 20, Main = 30, Game = 40, Settings = 100}

local function createScroll(parent)
	local s = create("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
		ScrollBarThickness = IS_TOUCH and 2 or 3, ScrollBarImageColor3 = COLORS.Accent,
		AutomaticCanvasSize = Enum.AutomaticSize.Y, CanvasSize = UDim2.new(),
		ScrollingDirection = Enum.ScrollingDirection.Y, Parent = parent,
	})
	create("UIListLayout", {Padding = UDim.new(0, 10), SortOrder = Enum.SortOrder.LayoutOrder, Parent = s})
	create("UIPadding", {PaddingBottom = UDim.new(0, 8), Parent = s})
	return s
end

local function setActiveTab(name)
	if not tabs[name] then return end
	activeTab = name
	for n, t in pairs(tabs) do
		local on = (n == name)
		t.Page.Visible = on
		t.Line.Visible = on
		t.Active = on
		tween(t.Btn, 0.2, {BackgroundTransparency = on and 0.4 or 1})
		tween(t.Icon, 0.2, {ImageColor3 = on and COLORS.Accent or COLORS.TextDark})
		tween(t.Label, 0.2, {TextColor3 = on and COLORS.TextMain or COLORS.TextDark})
	end
	HeaderTitle.Text = name
	HeaderIcon.Image = tabs[name].IconId
end

local function createTab(name, iconId, useScroll, order)
	local btn = create("TextButton", {
		Size = UDim2.new(1, 0, 0, 38), BackgroundColor3 = COLORS.SubTabBg, BackgroundTransparency = 1,
		Text = "", AutoButtonColor = false, LayoutOrder = order or nextOrder(TabsContainer), Parent = TabsContainer,
	})
	corner(btn, 8)
	local line = create("Frame", {
		Size = UDim2.fromOffset(3, 18), Position = UDim2.new(0, 0, 0.5, -9),
		BackgroundColor3 = COLORS.Accent, BorderSizePixel = 0, Visible = false, Parent = btn,
	})
	corner(line, 2)
	local icon = create("ImageLabel", {
		Size = UDim2.fromOffset(18, 18), Position = UDim2.new(0, 12, 0.5, -9),
		BackgroundTransparency = 1, Image = iconId, ImageColor3 = COLORS.TextDark, Parent = btn,
	})
	local lbl = label({Size = UDim2.new(1, -40, 1, 0), Position = UDim2.new(0, 38, 0, 0), Text = name,
		TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = btn})

	local page = create("Frame", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, Parent = PagesContainer})
	local content = page
	if useScroll then content = createScroll(page) end

	tabs[name] = {Btn = btn, Page = page, Icon = icon, Line = line, Label = lbl, IconId = iconId, Active = false}
	btn.MouseButton1Click:Connect(function()
		if activeTab ~= name then Sfx.Play("Click") end
		setActiveTab(name)
	end)
	btn.MouseEnter:Connect(function() if not tabs[name].Active then tween(btn, 0.15, {BackgroundTransparency = 0.8}) end end)
	btn.MouseLeave:Connect(function() if not tabs[name].Active then tween(btn, 0.15, {BackgroundTransparency = 1}) end end)
	return content
end

-- Grupo de sub-abas com barra rolável (cabe no celular)
local function createSubGroup(tabName, page)
	local bar = create("ScrollingFrame", {
		Size = UDim2.new(1, 0, 0, 34), BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 0,
		CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.X,
		ScrollingDirection = Enum.ScrollingDirection.X, Parent = page,
	})
	create("UIListLayout", {FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder,
		Padding = UDim.new(0, 8), Parent = bar})
	local holder = create("Frame", {
		Size = UDim2.new(1, 0, 1, -42), Position = UDim2.new(0, 0, 0, 42), BackgroundTransparency = 1, Parent = page,
	})
	local group = {Subs = {}, Active = nil}

	function group.Set(name)
		if not group.Subs[name] then return end
		group.Active = name
		for n, s in pairs(group.Subs) do
			local on = (n == name)
			s.Page.Visible = on
			tween(s.Btn, 0.2, {BackgroundColor3 = on and COLORS.SubTabActive or COLORS.SubTabBg})
			tween(s.Icon, 0.2, {ImageColor3 = on and COLORS.Accent or COLORS.TextDark})
			tween(s.Label, 0.2, {TextColor3 = on and COLORS.TextMain or COLORS.TextDark})
		end
	end

	function group.Add(name, iconId)
		local btn = create("TextButton", {
			Size = UDim2.fromOffset(96, 34), BackgroundColor3 = COLORS.SubTabBg, Text = "",
			AutoButtonColor = false, LayoutOrder = nextOrder(bar), Parent = bar,
		})
		corner(btn, 8)
		local icon = create("ImageLabel", {
			Size = UDim2.fromOffset(14, 14), Position = UDim2.new(0, 10, 0.5, -7),
			BackgroundTransparency = 1, Image = iconId, ImageColor3 = COLORS.TextDark, Parent = btn,
		})
		local lbl = label({Size = UDim2.new(1, -30, 1, 0), Position = UDim2.new(0, 30, 0, 0), Text = name,
			TextSize = 11, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = btn})
		local pg = create("Frame", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, Parent = holder})
		local scroll = createScroll(pg)
		group.Subs[name] = {Btn = btn, Page = pg, Icon = icon, Label = lbl}
		btn.MouseButton1Click:Connect(function()
			if group.Active ~= name then Sfx.Play("Click") end
			group.Set(name)
		end)
		if not group.Active then group.Set(name) end
		return scroll
	end

	groups[tabName] = group
	return group
end

-- ==========================================
-- 6. NOTIFICAÇÕES
-- ==========================================
local Notify
do
	local holder = create("Frame", {
		Name = "Notifications", Size = UDim2.new(0, IS_TOUCH and 200 or 250, 1, -20),
		Position = UDim2.new(1, IS_TOUCH and -212 or -262, 0, 10), BackgroundTransparency = 1, ZIndex = 50, Parent = ScreenGui,
	})
	create("UIListLayout", {
		VerticalAlignment = Enum.VerticalAlignment.Bottom, SortOrder = Enum.SortOrder.LayoutOrder,
		Padding = UDim.new(0, 8), Parent = holder,
	})
	local active = 0
	-- noSound = true quando outro som (ex.: toggle) já tocou
	Notify = function(title, text, duration, noSound)
		if not notificationsEnabled or active >= 3 then return end
		active = active + 1
		duration = duration or 2.5
		if not noSound then Sfx.Play("Notify", 0.15) end
		local wrapper = create("Frame", {
			Size = UDim2.new(1, 0, 0, 50), BackgroundTransparency = 1, ZIndex = 50,
			LayoutOrder = nextOrder(holder), Parent = holder,
		})
		local card = create("CanvasGroup", {
			Size = UDim2.fromScale(1, 1), Position = UDim2.new(1.2, 0, 0, 0),
			BackgroundColor3 = COLORS.Card, BorderSizePixel = 0, Parent = wrapper,
		})
		corner(card, 10)
		stroke(card, COLORS.Accent, 1, 0.5)
		local bar = create("Frame", {Size = UDim2.new(0, 3, 1, -16), Position = UDim2.new(0, 8, 0, 8),
			BackgroundColor3 = COLORS.Accent, BorderSizePixel = 0, Parent = card})
		corner(bar, 2)
		label({Size = UDim2.new(1, -28, 0, 16), Position = UDim2.new(0, 20, 0, 8), Text = title,
			Font = Enum.Font.GothamBold, TextSize = 12, TextTruncate = Enum.TextTruncate.AtEnd, Parent = card})
		label({Size = UDim2.new(1, -28, 0, 14), Position = UDim2.new(0, 20, 0, 26), Text = text,
			TextSize = 10, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = card})
		tween(card, 0.35, {Position = UDim2.new(0, 0, 0, 0)}, Enum.EasingStyle.Back)
		task.delay(duration, function()
			if card.Parent then
				tween(card, 0.3, {GroupTransparency = 1, Position = UDim2.new(0.3, 0, 0, 0)})
				task.wait(0.32)
			end
			active = math.max(0, active - 1)
			if wrapper.Parent then wrapper:Destroy() end
		end)
	end
end

-- ==========================================
-- 7. COMPONENTES (+ registro para a busca)
-- ==========================================
local sections = {}

local function safeCall(fn, ...)
	local ok, err = pcall(fn, ...)
	if not ok then warn("[ShadowHub] " .. tostring(err)) end
end

local function AddSection(parent, title, tab, sub)
	local frame = create("Frame", {
		Size = UDim2.new(1, -8, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
		BackgroundColor3 = COLORS.Card, BorderSizePixel = 0, LayoutOrder = nextOrder(parent), Parent = parent,
	})
	corner(frame, 12)
	stroke(frame, COLORS.Stroke, 1)
	create("UIListLayout", {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 2), Parent = frame})
	create("UIPadding", {PaddingTop = UDim.new(0, 8), PaddingBottom = UDim.new(0, 10), Parent = frame})
	local header = label({Size = UDim2.new(1, 0, 0, 20), Text = string.upper(title), TextSize = 10,
		Font = Enum.Font.GothamBold, TextColor3 = COLORS.Accent, LayoutOrder = nextOrder(frame), Parent = frame})
	create("UIPadding", {PaddingLeft = UDim.new(0, 14), Parent = header})
	local sec = {Frame = frame, Tab = tab, Sub = sub, Rows = {}}
	if tab then table.insert(sections, sec) end
	return sec
end

local function registerRow(sec, row, title)
	table.insert(sec.Rows, {Frame = row, Title = string.lower(title)})
end

-- quiet = true: sem som e sem pop-up ao ligar/desligar (usado por Auto Rebirth / Auto Upgrade)
local function AddToggle(sec, title, desc, default, callback, quiet)
	local h = desc and 46 or 36
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, h), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	-- Linha inteira clicável (melhor no toque)
	local rowBtn = create("TextButton", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "",
		AutoButtonColor = false, Parent = row})
	label({Size = UDim2.new(1, -84, 0, desc and 20 or h), Position = UDim2.new(0, 14, 0, desc and 4 or 0),
		Text = title, TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	if desc then
		label({Size = UDim2.new(1, -84, 0, 16), Position = UDim2.new(0, 14, 0, 24), Text = desc,
			TextSize = 10, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	end
	local switch = create("Frame", {
		Size = UDim2.fromOffset(42, 22), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
		BackgroundColor3 = default and COLORS.Accent or COLORS.SwitchOff, Parent = row,
	})
	corner(switch, 11)
	local knob = create("Frame", {
		Size = UDim2.fromOffset(16, 16),
		Position = default and UDim2.new(1, -19, 0.5, -8) or UDim2.new(0, 3, 0.5, -8),
		BackgroundColor3 = Color3.new(1, 1, 1), Parent = switch,
	})
	corner(knob, 8)

	local state = default and true or false
	local control = {}
	function control.Set(newState, silent)
		state = newState and true or false
		tween(knob, 0.2, {Position = state and UDim2.new(1, -19, 0.5, -8) or UDim2.new(0, 3, 0.5, -8)})
		tween(switch, 0.2, {BackgroundColor3 = state and COLORS.Accent or COLORS.SwitchOff})
		if not silent then
			if not quiet then Sfx.Play(state and "On" or "Off") end
			safeCall(callback, state)
			if not quiet then Notify(title, state and "Enabled" or "Disabled", 1.5, true) end
		end
	end
	function control.Get() return state end
	rowBtn.MouseButton1Click:Connect(function() control.Set(not state) end)

	registerRow(sec, row, title)
	return control
end

local function AddSlider(sec, title, minVal, maxVal, default, step, unit, callback)
	local decimals = step >= 1 and 0 or (step >= 0.1 and 1 or 2)
	local function fmt(v) return string.format("%." .. decimals .. "f", v) .. unit end
	local rowH = IS_TOUCH and 60 or 54

	local row = create("Frame", {Size = UDim2.new(1, 0, 0, rowH), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(1, -90, 0, 20), Position = UDim2.new(0, 14, 0, 4), Text = title,
		TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	local valBox = create("Frame", {Size = UDim2.fromOffset(56, 20), AnchorPoint = Vector2.new(1, 0),
		Position = UDim2.new(1, -14, 0, 4), BackgroundColor3 = COLORS.CardHeader, Parent = row})
	corner(valBox, 6)
	local valLabel = label({Size = UDim2.fromScale(1, 1), Text = fmt(default), TextSize = 10,
		Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Center, Parent = valBox})

	local hit = create("Frame", {Size = UDim2.new(1, -28, 0, IS_TOUCH and 30 or 24), Position = UDim2.new(0, 14, 0, 26),
		BackgroundTransparency = 1, Active = true, Parent = row})
	local trackBar = create("Frame", {Size = UDim2.new(1, 0, 0, 4), AnchorPoint = Vector2.new(0, 0.5),
		Position = UDim2.new(0, 0, 0.5, 0), BackgroundColor3 = COLORS.SwitchOff, BorderSizePixel = 0, Parent = hit})
	corner(trackBar, 2)
	local fill = create("Frame", {Size = UDim2.fromScale((default - minVal) / (maxVal - minVal), 1),
		BackgroundColor3 = COLORS.Accent, BorderSizePixel = 0, Parent = trackBar})
	corner(fill, 2)
	local thumb = create("Frame", {Size = UDim2.fromOffset(12, 12), AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromScale(1, 0.5), BackgroundColor3 = COLORS.AccentGlow, Parent = fill})
	corner(thumb, 6)

	local last = default
	local control = {}
	function control.Set(v, fire)
		v = math.clamp(v, minVal, maxVal)
		v = minVal + math.floor((v - minVal) / step + 0.5) * step
		v = math.clamp(math.floor(v * 1000 + 0.5) / 1000, minVal, maxVal)
		fill.Size = UDim2.fromScale((v - minVal) / (maxVal - minVal), 1)
		valLabel.Text = fmt(v)
		if fire and v ~= last then
			last = v
			Sfx.Play("Slide", 0.07)
			safeCall(callback, v)
		end
		last = v
	end

	local dragging, dragInput = false, nil
	local function updateFromX(x)
		local w = hit.AbsoluteSize.X
		if w <= 0 then return end
		local pos = math.clamp((x - hit.AbsolutePosition.X) / w, 0, 1)
		control.Set(minVal + (maxVal - minVal) * pos, true)
	end

	hit.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging, dragInput = true, input
			local scroller = row:FindFirstAncestorWhichIsA("ScrollingFrame")
			if scroller then scroller.ScrollingEnabled = false end
			tween(thumb, 0.15, {Size = UDim2.fromOffset(16, 16)})
			updateFromX(input.Position.X)
		end
	end)
	track(UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input == dragInput) then
			updateFromX(input.Position.X)
		end
	end))
	track(UserInputService.InputEnded:Connect(function(input)
		if dragging and (input == dragInput or input.UserInputType == Enum.UserInputType.MouseButton1) then
			dragging = false
			local scroller = row:FindFirstAncestorWhichIsA("ScrollingFrame")
			if scroller then scroller.ScrollingEnabled = true end
			tween(thumb, 0.15, {Size = UDim2.fromOffset(12, 12)})
		end
	end))

	registerRow(sec, row, title)
	return control
end

local function AddButton(sec, title, callback, danger)
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, IS_TOUCH and 44 or 40), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	local base = danger and COLORS.Danger or COLORS.CardHeader
	local over = danger and Color3.fromRGB(225, 80, 100) or COLORS.CardHover
	local btn = create("TextButton", {
		Size = UDim2.new(1, -28, 0, 34), Position = UDim2.new(0, 14, 0.5, -17), BackgroundColor3 = base,
		Text = title, TextColor3 = COLORS.TextMain, TextSize = 12, Font = Enum.Font.GothamBold,
		AutoButtonColor = false, Parent = row,
	})
	corner(btn, 8)
	stroke(btn, danger and COLORS.Danger or COLORS.Stroke, 1)
	btn.MouseEnter:Connect(function() tween(btn, 0.15, {BackgroundColor3 = over}) end)
	btn.MouseLeave:Connect(function() tween(btn, 0.15, {BackgroundColor3 = base}) end)
	btn.MouseButton1Down:Connect(function() tween(btn, 0.08, {Size = UDim2.new(1, -34, 0, 32)}) end)
	btn.MouseButton1Up:Connect(function() tween(btn, 0.12, {Size = UDim2.new(1, -28, 0, 34)}) end)
	btn.MouseButton1Click:Connect(function()
		Sfx.Play(danger and "Error" or "Click")
		safeCall(callback)
	end)
	registerRow(sec, row, title)
	return btn
end

local function AddInput(sec, title, placeholder, default, callback)
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 40), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(1, -190, 1, 0), Position = UDim2.new(0, 14, 0, 0), Text = title,
		TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	local boxFrame = create("Frame", {
		Size = UDim2.fromOffset(160, 28), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
		BackgroundColor3 = COLORS.CardHeader, Parent = row,
	})
	corner(boxFrame, 6)
	local bs = stroke(boxFrame, COLORS.Stroke, 1)
	local box = create("TextBox", {
		Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 6, 0, 0), BackgroundTransparency = 1,
		Text = default or "", PlaceholderText = placeholder or "", PlaceholderColor3 = COLORS.TextDark,
		TextColor3 = COLORS.TextMain, TextSize = 11, Font = Enum.Font.GothamMedium,
		TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false, ClipsDescendants = true, Parent = boxFrame,
	})
	box.Focused:Connect(function() tween(bs, 0.2, {Color = COLORS.Accent}) end)
	box.FocusLost:Connect(function()
		tween(bs, 0.2, {Color = COLORS.Stroke})
		Sfx.Play("Click")
		safeCall(callback, box.Text)
	end)
	registerRow(sec, row, title)
	return box
end

local function AddInfo(sec, title, value)
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 26), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(0.4, 0, 1, 0), Position = UDim2.new(0, 14, 0, 0), Text = title,
		TextSize = 11, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	local val = label({Size = UDim2.new(0.6, -28, 1, 0), Position = UDim2.new(0.4, 0, 0, 0), Text = tostring(value),
		TextSize = 11, Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Right,
		TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	registerRow(sec, row, title)
	return val
end

local function AddText(sec, text)
	local l = label({Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, Text = text,
		TextSize = 11, TextColor3 = COLORS.TextDark, TextWrapped = true, TextYAlignment = Enum.TextYAlignment.Top,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	create("UIPadding", {PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14), Parent = l})
	return l
end

-- ==========================================
-- 8. BUSCA (debounce + navegação até o resultado)
-- ==========================================
do
	local token = 0
	local function applySearch()
		local q = (string.lower(SearchBox.Text):gsub("^%s*(.-)%s*$", "%1"))
		local firstSec, currentHas, anyMatch = nil, false, false
		for _, sec in ipairs(sections) do
			local any = false
			for _, r in ipairs(sec.Rows) do
				local match = (q == "") or (string.find(r.Title, q, 1, true) ~= nil)
				if r.Frame.Visible ~= match then r.Frame.Visible = match end
				if match then any = true end
			end
			local vis = (q == "") or any
			if sec.Frame.Visible ~= vis then sec.Frame.Visible = vis end
			if q ~= "" and any then
				anyMatch = true
				firstSec = firstSec or sec
				if sec.Tab == activeTab then
					local g = groups[sec.Tab]
					if sec.Sub == nil or (g and g.Active == sec.Sub) then currentHas = true end
				end
			end
		end
		NoResults.Visible = (q ~= "" and not anyMatch)
		if q ~= "" and firstSec and not currentHas then
			setActiveTab(firstSec.Tab)
			if firstSec.Sub and groups[firstSec.Tab] then groups[firstSec.Tab].Set(firstSec.Sub) end
		end
	end
	SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
		token = token + 1
		local my = token
		task.delay(0.12, function()
			if my == token then applySearch() end
		end)
	end)
end

-- ==========================================
-- 9. NÚCLEO: MOVIMENTO, VISUAIS, PERFORMANCE, ESP, HOOK
-- ==========================================

-- ---------- 9.1 Movimento ----------
function Mv.restoreSpeed()
	local hum = getHum()
	if hum and Original.Speed then hum.WalkSpeed = Original.Speed end
	Original.Speed = nil
end

function Mv.restoreJump()
	local hum = getHum()
	if hum and Original.Jump then
		if Original.JumpIsPower then hum.JumpPower = Original.Jump else hum.JumpHeight = Original.Jump end
	end
	Original.Jump = nil
end

function Mv.rebuildNoclip()
	table.clear(Mv.parts)
	local c = LocalPlayer.Character
	if not c then return end
	for _, p in ipairs(c:GetDescendants()) do
		if p:IsA("BasePart") then Mv.parts[#Mv.parts + 1] = p end
	end
end

function Mv.restoreNoclip()
	for part in pairs(Mv.touched) do
		if part.Parent then part.CanCollide = true end
	end
	Mv.touched = {}
	table.clear(Mv.parts)
end

function Mv.stopFly()
	if Mv.fly then
		for _, o in pairs(Mv.fly) do if o then o:Destroy() end end
		Mv.fly = nil
	end
	Mv.vel = Vector3.zero
	local hum = getHum()
	if hum then
		hum.PlatformStand = false
		pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
	end
end

function Mv.startFly()
	local root, hum = getRoot(), getHum()
	if not root or not hum then return end
	Mv.stopFly()
	local bv = Instance.new("BodyVelocity")
	bv.MaxForce = Vector3.new(1e9, 1e9, 1e9)
	bv.Velocity = Vector3.zero
	bv.Parent = root
	local bg = Instance.new("BodyGyro")
	bg.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
	bg.P = 9e4
	bg.CFrame = root.CFrame
	bg.Parent = root
	hum.PlatformStand = true
	Mv.fly = {bv = bv, bg = bg}
end

function Mv.moveVector()
	if Mv.controls then
		local ok, v = pcall(function() return Mv.controls:GetMoveVector() end)
		if ok and v then return v end
	end
	local hum = getHum()
	local cam = workspace.CurrentCamera
	if hum and cam then
		local md = cam.CFrame:VectorToObjectSpace(hum.MoveDirection)
		return Vector3.new(md.X, 0, md.Z)
	end
	return Vector3.zero
end

-- ---------- 9.2 Visuais ----------
local Vis = {}
function Vis.restoreFullbright()
	if Vis.fb then
		for k, v in pairs(Vis.fb) do pcall(function() Lighting[k] = v end) end
		Vis.fb = nil
	end
end
function Vis.restoreFog()
	if Vis.fog then
		pcall(function() Lighting.FogEnd = Vis.fog.FogEnd; Lighting.FogStart = Vis.fog.FogStart end)
		for atm, d in pairs(Vis.fog.Atmos) do if atm.Parent then atm.Density = d end end
		Vis.fog = nil
	end
end

-- ---------- 9.3 Performance: Anti-Lag, FPS Boost, Guard ----------
local alSet = setmetatable({}, {__mode = "k"})
local alPost = {}
local alConn
local ultraStore = setmetatable({}, {__mode = "k"})
local ultraConn

local function killEffect(d)
	if d:IsA("ParticleEmitter") or d:IsA("Trail") or d:IsA("Beam") or d:IsA("Fire") or d:IsA("Smoke") or d:IsA("Sparkles") then
		if d.Enabled and not alSet[d] then
			alSet[d] = true
			d.Enabled = false
		end
	end
end

-- Remove efeitos pesados (partículas, rastros, blur, bloom...) NO SEU CLIENTE
local function setAntiLag(on)
	if on == Perf.alOn then return end
	Perf.alOn = on
	if on then
		task.spawn(function()
			local i = 0
			for _, d in ipairs(workspace:GetDescendants()) do
				if not Perf.alOn then break end
				killEffect(d)
				i = i + 1
				if i % 500 == 0 then task.wait() end
			end
		end)
		alConn = workspace.DescendantAdded:Connect(function(d) if Perf.alOn then killEffect(d) end end)
		for _, c in ipairs(Lighting:GetChildren()) do
			if c:IsA("BlurEffect") or c:IsA("BloomEffect") or c:IsA("SunRaysEffect") or c:IsA("DepthOfFieldEffect") then
				if c.Enabled then alPost[c] = true; c.Enabled = false end
			end
		end
	else
		if alConn then alConn:Disconnect(); alConn = nil end
		for d in pairs(alSet) do if d.Parent then d.Enabled = true end end
		alSet = setmetatable({}, {__mode = "k"})
		for c in pairs(alPost) do if c.Parent then c.Enabled = true end end
		alPost = {}
	end
end

local function ultraApply(d)
	if d:IsA("Terrain") then return end
	if d:IsA("BasePart") then
		if not ultraStore[d] then ultraStore[d] = {d.Material, d.Reflectance} end
		d.Material = Enum.Material.SmoothPlastic
		d.Reflectance = 0
	elseif d:IsA("Decal") or d:IsA("Texture") then
		if not ultraStore[d] then ultraStore[d] = {d.Transparency} end
		d.Transparency = 1
	end
end

-- Ultra Boost: tira texturas/materiais (mais FPS, visual simples)
local function setUltra(on)
	if on == Perf.ultraOn then return end
	Perf.ultraOn = on
	if on then
		task.spawn(function()
			local i = 0
			for _, d in ipairs(workspace:GetDescendants()) do
				if not Perf.ultraOn then break end
				pcall(ultraApply, d)
				i = i + 1
				if i % 400 == 0 then task.wait() end
			end
		end)
		ultraConn = workspace.DescendantAdded:Connect(function(d) if Perf.ultraOn then pcall(ultraApply, d) end end)
	else
		if ultraConn then ultraConn:Disconnect(); ultraConn = nil end
		for d, v in pairs(ultraStore) do
			if d.Parent then
				pcall(function()
					if d:IsA("BasePart") then d.Material = v[1]; d.Reflectance = v[2]
					else d.Transparency = v[1] end
				end)
			end
		end
		ultraStore = setmetatable({}, {__mode = "k"})
	end
end

-- FPS Boost: qualidade mínima, sem sombras e sem decoração de terreno
local function setLowGfx(on)
	State.LowGfx = on
	if on then
		if not Perf.gfx then
			Perf.gfx = {}
			pcall(function() Perf.gfx.Quality = settings().Rendering.QualityLevel end)
			pcall(function() Perf.gfx.Shadows = Lighting.GlobalShadows end)
			pcall(function() Perf.gfx.Decoration = workspace.Terrain.Decoration end)
			pcall(function() Perf.gfx.Wave = workspace.Terrain.WaterWaveSize end)
		end
		pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
		pcall(function() Lighting.GlobalShadows = false end)
		pcall(function() workspace.Terrain.Decoration = false end)
		pcall(function() workspace.Terrain.WaterWaveSize = 0 end)
	elseif Perf.gfx then
		local g = Perf.gfx
		if g.Quality then pcall(function() settings().Rendering.QualityLevel = g.Quality end) end
		if g.Shadows ~= nil then pcall(function() Lighting.GlobalShadows = g.Shadows end) end
		if g.Decoration ~= nil then pcall(function() workspace.Terrain.Decoration = g.Decoration end) end
		if g.Wave ~= nil then pcall(function() workspace.Terrain.WaterWaveSize = g.Wave end) end
		Perf.gfx = nil
	end
end

-- Modo Lite: o próprio hub reduz seu custo (neve off, ESP e loops mais lentos)
local function setLite(on)
	if Perf.Lite == on then return end
	Perf.Lite = on
	SnowContainer.Visible = State.Snow and not on and isOpen
end

-- Verificado 1x por segundo com o FPS medido
local function guardTick(fps)
	Perf.Fps = fps
	if not Perf.Guard or fps < 5 then
		Perf.lowSec, Perf.critSec, Perf.goodSec = 0, 0, 0
		Perf.Status = Perf.Guard and "Paused" or "Off"
		return
	end
	if fps < Perf.LowFps then Perf.lowSec = Perf.lowSec + 1 else Perf.lowSec = 0 end
	if fps < Perf.CritFps then Perf.critSec = Perf.critSec + 1 else Perf.critSec = 0 end
	if fps >= Perf.LowFps + 12 then Perf.goodSec = Perf.goodSec + 1 else Perf.goodSec = 0 end

	if Perf.lowSec >= 3 and not Perf.Lite then
		setLite(true)
		if not Perf.alOn then
			setAntiLag(true)
			Perf.autoAL = true
			if PerfUI.al then PerfUI.al.Set(true, true) end
		end
		Notify("Performance Guard", "FPS low: Anti-Lag enabled", 3)
	end
	if Perf.critSec >= 3 and not State.LowGfx then
		setLowGfx(true)
		Perf.autoFB = true
		if PerfUI.gfx then PerfUI.gfx.Set(true, true) end
		Notify("Performance Guard", "FPS critical: FPS Boost enabled", 3)
	end
	if Perf.AutoRestore and Perf.goodSec >= 30 and Perf.flaps < 2 and (Perf.autoAL or Perf.autoFB or Perf.Lite) then
		if Perf.autoFB then
			setLowGfx(false)
			Perf.autoFB = false
			if PerfUI.gfx then PerfUI.gfx.Set(false, true) end
		end
		if Perf.autoAL then
			setAntiLag(false)
			Perf.autoAL = false
			if PerfUI.al then PerfUI.al.Set(false, true) end
		end
		setLite(false)
		Perf.flaps = Perf.flaps + 1
		Perf.goodSec = 0
		Notify("Performance Guard", "FPS recovered: settings restored", 3)
	end
	Perf.Status = Perf.Lite and "Boosting" or "Monitoring"
end

-- ---------- 9.4 Utilitários ----------
local afkConn, rejoinConn

local function setAntiAfk(on)
	State.AntiAfk = on
	if on then
		if not afkConn then
			afkConn = LocalPlayer.Idled:Connect(function()
				pcall(function()
					local vu = game:GetService("VirtualUser")
					vu:CaptureController()
					vu:ClickButton2(Vector2.new())
				end)
			end)
		end
	elseif afkConn then
		afkConn:Disconnect()
		afkConn = nil
	end
end

local function setNoRender(on)
	State.NoRender = on
	pcall(function() RunService:Set3dRenderingEnabled(not on) end)
end

local function setAutoRejoin(on)
	State.AutoRejoin = on
	if on then
		if not rejoinConn then
			local busy = false
			rejoinConn = GuiService.ErrorMessageChanged:Connect(function(msg)
				if State.AutoRejoin and not busy and msg and msg ~= "" then
					busy = true
					task.wait(3)
					pcall(function() TeleportService:Teleport(game.PlaceId, LocalPlayer) end)
					task.delay(10, function() busy = false end)
				end
			end)
		end
	elseif rejoinConn then
		rejoinConn:Disconnect()
		rejoinConn = nil
	end
end

-- Hook de remotes (usado só pelo aprendizado do merge; inerte fora disso)
local function installHook()
	if env.__ShadowHookInstalled then return true end
	if not (hookmetamethod and getnamecallmethod) then return false end
	local wrap = newcclosure or function(f) return f end
	local old
	local ok = pcall(function()
		old = hookmetamethod(game, "__namecall", wrap(function(self, ...)
			local m = getnamecallmethod()
			local h = env.__ShadowHookFn
			if h and (m == "FireServer" or m == "InvokeServer") and (not checkcaller or not checkcaller()) then
				pcall(h, self, m, ...)
				if setnamecallmethod then setnamecallmethod(m) end
			end
			return old(self, ...)
		end))
	end)
	if ok then env.__ShadowHookInstalled = true end
	return ok
end

-- ---------- 9.5 ESP (motor genérico com provedores) ----------
local ESPFolder = create("Folder", {Name = "ShadowESP", Parent = ParentGui})
local ESP = {Providers = {}, Enabled = {}, Entries = {}, MaxDist = 800, Limit = 40}

function ESP.Register(name, fn)
	ESP.Providers[name] = fn
	ESP.Enabled[name] = false
	ESP.Entries[name] = {}
end

function ESP.Clear(name)
	for _, e in pairs(ESP.Entries[name] or {}) do
		if e.gui then e.gui:Destroy() end
		if e.hl then e.hl:Destroy() end
	end
	ESP.Entries[name] = {}
end

function ESP.SetEnabled(name, on)
	ESP.Enabled[name] = on and true or false
	if not on then ESP.Clear(name) end
end

function ESP.Tick()
	local cam = workspace.CurrentCamera
	if not cam then return end
	local camPos = cam.CFrame.Position
	local hlUsed = 0
	for name, fn in pairs(ESP.Providers) do
		if ESP.Enabled[name] then
			local ok, items = pcall(fn)
			local entries = ESP.Entries[name]
			local mark = {}
			if ok and items then
				for i = 1, math.min(#items, ESP.Limit) do
					local it = items[i]
					local part = it.part
					if part and part.Parent then
						local dist = (part.Position - camPos).Magnitude
						if dist <= ESP.MaxDist then
							local e = entries[part]
							if not e then
								local gui = create("BillboardGui", {
									Size = UDim2.fromOffset(130, 34), StudsOffset = Vector3.new(0, 3, 0), AlwaysOnTop = true,
									LightInfluence = 0, MaxDistance = ESP.MaxDist + 50, Adornee = part, Parent = ESPFolder,
								})
								local lbl = create("TextLabel", {
									Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Font = Enum.Font.GothamBold,
									TextSize = 12, TextColor3 = it.color, TextStrokeTransparency = 0.3,
									TextStrokeColor3 = Color3.new(0, 0, 0), Text = "", Parent = gui,
								})
								e = {gui = gui, lbl = lbl}
								entries[part] = e
							end
							mark[part] = true
							local txt = it.text .. "\n" .. math.floor(dist) .. "m"
							if e.txt ~= txt then e.txt = txt; e.lbl.Text = txt end
							if e.col ~= it.color then e.col = it.color; e.lbl.TextColor3 = it.color end
							if it.hl and it.hl.Parent and hlUsed < 28 then
								hlUsed = hlUsed + 1
								if not e.hl then
									e.hl = create("Highlight", {
										Adornee = it.hl, FillColor = it.color, OutlineColor = Color3.new(1, 1, 1),
										FillTransparency = 0.55, OutlineTransparency = 0,
										DepthMode = Enum.HighlightDepthMode.AlwaysOnTop, Parent = ESPFolder,
									})
								else
									if e.hl.Adornee ~= it.hl then e.hl.Adornee = it.hl end
									if e.hl.FillColor ~= it.color then e.hl.FillColor = it.color end
								end
							elseif e.hl then
								e.hl:Destroy()
								e.hl = nil
							end
						end
					end
				end
			end
			for part, e in pairs(entries) do
				if not mark[part] then
					if e.gui then e.gui:Destroy() end
					if e.hl then e.hl:Destroy() end
					entries[part] = nil
				end
			end
		end
	end
end

ESP.Register("Players", function()
	local out = {}
	for _, pl in ipairs(Players:GetPlayers()) do
		if pl ~= LocalPlayer then
			local ch = pl.Character
			local root = ch and ch:FindFirstChild("HumanoidRootPart")
			if root then
				out[#out + 1] = {part = root, hl = ch, text = pl.DisplayName,
					color = (pl.Team and pl.TeamColor.Color) or COLORS.AccentGlow}
			end
		end
	end
	return out
end)

-- ==========================================
-- 10. ABAS PRINCIPAIS (universais: aparecem em qualquer jogo)
-- ==========================================

-- ---------- 10.1 HOME ----------
do
	local HomeScroll = createTab("Home", ICONS.Home, true, ORDER.Home)

	local card = create("Frame", {
		Size = UDim2.new(1, -8, 0, 72), BackgroundColor3 = COLORS.Card, BorderSizePixel = 0,
		LayoutOrder = nextOrder(HomeScroll), Parent = HomeScroll,
	})
	corner(card, 12)
	stroke(card, COLORS.Stroke, 1)
	Refs.homeAvatar = create("ImageLabel", {
		Size = UDim2.fromOffset(48, 48), Position = UDim2.new(0, 14, 0.5, -24),
		BackgroundColor3 = COLORS.CardHeader, Image = "", Parent = card,
	})
	corner(Refs.homeAvatar, 24)
	stroke(Refs.homeAvatar, COLORS.Accent, 1.5, 0.2)
	label({Size = UDim2.new(1, -90, 0, 14), Position = UDim2.new(0, 74, 0, 14), Text = "Welcome back,",
		TextSize = 11, TextColor3 = COLORS.TextDark, Parent = card})
	label({Size = UDim2.new(1, -90, 0, 20), Position = UDim2.new(0, 74, 0, 32),
		Text = LocalPlayer.DisplayName .. " (@" .. LocalPlayer.Name .. ")", Font = Enum.Font.GothamBold,
		TextSize = 14, TextTruncate = Enum.TextTruncate.AtEnd, Parent = card})

	task.spawn(function()
		local ok, img = pcall(function()
			return (Players:GetUserThumbnailAsync(LocalPlayer.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100))
		end)
		if ok and img then
			AvatarImg.Image = img
			Refs.homeAvatar.Image = img
		end
	end)

	local grid = create("Frame", {
		Size = UDim2.new(1, -8, 0, 54), BackgroundTransparency = 1, LayoutOrder = nextOrder(HomeScroll), Parent = HomeScroll,
	})
	create("UIGridLayout", {CellSize = UDim2.new(0.235, 0, 1, 0), CellPadding = UDim2.new(0.02, 0, 0, 0),
		SortOrder = Enum.SortOrder.LayoutOrder, Parent = grid})
	local function statBox(title, initial)
		local box = create("Frame", {BackgroundColor3 = COLORS.Card, BorderSizePixel = 0,
			LayoutOrder = nextOrder(grid), Parent = grid})
		corner(box, 10)
		stroke(box, COLORS.Stroke, 1)
		label({Size = UDim2.new(1, -16, 0, 16), Position = UDim2.new(0, 10, 0, 8), Text = title,
			TextSize = 10, TextColor3 = COLORS.TextDark, Parent = box})
		return label({Size = UDim2.new(1, -16, 0, 20), Position = UDim2.new(0, 10, 0, 26), Text = initial,
			TextSize = 13, Font = Enum.Font.GothamBold, Parent = box})
	end
	Refs.players = statBox("Players", #Players:GetPlayers() .. "/" .. Players.MaxPlayers)
	Refs.session = statBox("Session", "00m 00s")
	Refs.fps     = statBox("FPS", "--")
	Refs.ping    = statBox("Ping", "--")

	local gameSec = AddSection(HomeScroll, "Game")
	Refs.gameName = AddInfo(gameSec, "Name", "Loading...")
	AddInfo(gameSec, "Place ID", tostring(game.PlaceId))
	Refs.age = AddInfo(gameSec, "Server age", "--")
	Refs.gameId = AddInfo(gameSec, "Game ID", tostring(game.GameId))
	Refs.gameScript = AddInfo(gameSec, "Game script", "Detecting...")
	Refs.device = AddInfo(gameSec, "Mode", IS_TOUCH and "Mobile / Emulator" or "PC")
	AddButton(gameSec, "Copy Game Info (Name / PlaceId / GameId)", function()
		local text = string.format("Name: %s | PlaceId: %d | GameId: %d", Refs.gameName.Text, game.PlaceId, game.GameId)
		if clipboardFn then clipboardFn(text); Notify("Game Info", "Copied to clipboard", 2.5)
		else Notify("Game Info", text, 5) end
	end)

	local about = AddSection(HomeScroll, "About")
	AddText(about, "Dark Shadow Hub v4.2\nMenu: botão flutuante" .. (IS_TOUCH and "" or " ou tecla (padrão Right Shift)") .. ". Arraste a sidebar ou o cabeçalho para mover. Scripts de jogo aparecem sozinhos, só no jogo certo. Performance Guard ligado por padrão.")
end

-- ---------- 10.2 PLAYER ----------
do
	local scroll = createTab("Player", ICONS.Player, true, ORDER.Player)

	local move = AddSection(scroll, "Movement", "Player")
	AddToggle(move, "Custom WalkSpeed", "Overrides your walk speed", false, function(on)
		State.SpeedOn = on
		if not on then Mv.restoreSpeed() end
	end)
	AddSlider(move, "WalkSpeed", 16, 200, State.Speed, 1, "", function(v) State.Speed = v end)
	AddToggle(move, "Custom JumpPower", "Overrides your jump strength", false, function(on)
		State.JumpOn = on
		if not on then Mv.restoreJump() end
	end)
	AddSlider(move, "JumpPower", 50, 300, State.Jump, 1, "", function(v) State.Jump = v end)
	AddToggle(move, "Infinite Jump", "Jump again while in the air", false, function(on) State.InfJump = on end)
	AddToggle(move, "Noclip", "Walk through walls", false, function(on)
		State.Noclip = on
		if on then Mv.rebuildNoclip() else Mv.restoreNoclip() end
	end)

	local fly = AddSection(scroll, "Fly", "Player")
	AddToggle(fly, "Fly", "Space / Ctrl = up / down. Mobile: joystick + camera", false, function(on)
		State.Fly = on
		if on then Mv.startFly() else Mv.stopFly() end
	end)
	AddSlider(fly, "Fly Speed", 10, 250, State.FlySpeed, 1, "", function(v) State.FlySpeed = v end)

	local char = AddSection(scroll, "Character", "Player")
	AddButton(char, "Reset Character", function()
		local hum = getHum()
		if hum then hum.Health = 0 end
	end)
	AddButton(char, "Copy Position", function()
		local root = getRoot()
		if not root then Notify("Position", "Character not found", 2) return end
		local p = root.Position
		local text = string.format("%.1f, %.1f, %.1f", p.X, p.Y, p.Z)
		if clipboardFn then clipboardFn(text); Notify("Position", "Copied: " .. text, 2.5)
		else Notify("Position", text, 4) end
	end)
end

-- ---------- 10.3 MAIN (Visuals / Perf / Utility / Server) ----------
do
	local page = createTab("Main", ICONS.Main, false, ORDER.Main)
	local group = createSubGroup("Main", page)
	local visuals = group.Add("Visuals", ICONS.Merge)
	local perf    = group.Add("Perf", ICONS.Shield)
	local utility = group.Add("Utility", ICONS.Settings)
	local server  = group.Add("Server", ICONS.Server)

	-- Visuals
	local light = AddSection(visuals, "Lighting", "Main", "Visuals")
	AddToggle(light, "Fullbright", "Removes darkness and shadows", false, function(on)
		State.Fullbright = on
		if on then
			Vis.fb = Vis.fb or {Brightness = Lighting.Brightness, ClockTime = Lighting.ClockTime,
				GlobalShadows = Lighting.GlobalShadows, Ambient = Lighting.Ambient, OutdoorAmbient = Lighting.OutdoorAmbient}
		else
			Vis.restoreFullbright()
		end
	end)
	AddToggle(light, "Remove Fog", "Clears fog and atmosphere haze", false, function(on)
		State.NoFog = on
		if on then
			if not Vis.fog then
				Vis.fog = {FogEnd = Lighting.FogEnd, FogStart = Lighting.FogStart, Atmos = {}}
				for _, c in ipairs(Lighting:GetChildren()) do
					if c:IsA("Atmosphere") then Vis.fog.Atmos[c] = c.Density end
				end
			end
		else
			Vis.restoreFog()
		end
	end)

	local cam = AddSection(visuals, "Camera", "Main", "Visuals")
	AddToggle(cam, "Custom FOV", "Overrides the field of view", false, function(on)
		State.FovOn = on
		local c = workspace.CurrentCamera
		if on then
			if c then Original.Fov = Original.Fov or c.FieldOfView end
		else
			if c and Original.Fov then c.FieldOfView = Original.Fov end
			Original.Fov = nil
		end
	end)
	AddSlider(cam, "Field of View", 30, 120, State.Fov, 1, "°", function(v) State.Fov = v end)
	AddSlider(cam, "Max Zoom Distance", 10, 1000, math.clamp(math.floor(origMaxZoom), 10, 1000), 1, "", function(v)
		LocalPlayer.CameraMaxZoomDistance = v
	end)

	local espSec = AddSection(visuals, "ESP", "Main", "Visuals")
	Refs.espPlayers = AddToggle(espSec, "Players ESP", "Highlight + name + distance", false, function(on)
		ESP.SetEnabled("Players", on)
		if Refs.espPlayersMini then Refs.espPlayersMini.Set(on, true) end
	end)
	AddSlider(espSec, "ESP Max Distance", 100, 2000, ESP.MaxDist, 50, "", function(v) ESP.MaxDist = v end)

	-- Performance
	local guard = AddSection(perf, "Performance Guard", "Main", "Perf")
	AddToggle(guard, "Auto Guard", "Turns on Anti-Lag + FPS Boost if FPS drops", Perf.Guard, function(on) Perf.Guard = on end)
	AddSlider(guard, "Low FPS threshold", 10, 60, Perf.LowFps, 1, "", function(v) Perf.LowFps = v end)
	AddSlider(guard, "Critical FPS threshold", 5, 45, Perf.CritFps, 1, "", function(v) Perf.CritFps = v end)
	AddToggle(guard, "Auto restore", "Restores settings when FPS recovers", Perf.AutoRestore, function(on) Perf.AutoRestore = on end)
	Refs.perfFps = AddInfo(guard, "Current FPS", "--")
	Refs.perfState = AddInfo(guard, "Guard state", "Monitoring")

	local al = AddSection(perf, "Anti-Lag", "Main", "Perf")
	PerfUI.al = AddToggle(al, "Anti-Lag", "Removes particles, trails, beams, blur, bloom", false, function(on)
		Perf.autoAL = false
		setAntiLag(on)
	end)
	PerfUI.ultra = AddToggle(al, "Ultra Boost (textures)", "Flat materials, no decals. Big FPS gain", false, setUltra)
	AddText(al, "Anti-Lag age no seu cliente (efeitos e gráficos). Ele não consegue reduzir o lag do servidor em si, mas diminui o que o servidor te manda para desenhar.")

	local gfx = AddSection(perf, "FPS Boost", "Main", "Perf")
	PerfUI.gfx = AddToggle(gfx, "FPS Boost", "Lowest quality, no shadows, no terrain decor", false, function(on)
		Perf.autoFB = false
		setLowGfx(on)
	end)
	PerfUI.noRender = AddToggle(gfx, "Disable 3D Rendering", "Black screen, minimum CPU/GPU (AFK)", false, setNoRender)
	if setfpscap then
		AddSlider(gfx, "FPS Cap", 30, 240, 60, 5, "", function(v) pcall(setfpscap, v) end)
	end

	-- Utility
	local util = AddSection(utility, "Utility", "Main", "Utility")
	PerfUI.afk = AddToggle(util, "Anti-AFK", "Prevents the idle kick", false, setAntiAfk)
	AddToggle(util, "Auto Rejoin", "Rejoins automatically if you get disconnected", false, setAutoRejoin)

	-- Server
	local function copyText(text, name)
		if clipboardFn then
			clipboardFn(text)
			Notify(name, "Copied to clipboard", 2)
		else
			Notify(name, "Clipboard not supported", 2.5)
		end
	end
	local function rejoin()
		Notify("Rejoin", "Teleporting...", 3)
		local ok, err = pcall(function()
			if #Players:GetPlayers() <= 1 then
				TeleportService:Teleport(game.PlaceId, LocalPlayer)
			else
				TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LocalPlayer)
			end
		end)
		if not ok then Notify("Rejoin", "Failed: " .. tostring(err), 4) end
	end
	local hopping = false
	local function serverHop()
		if hopping then return end
		hopping = true
		Notify("Server Hop", "Searching for a server...", 3)
		task.spawn(function()
			local ok, result = pcall(function()
				local url = string.format("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100", game.PlaceId)
				return game:GetService("HttpService"):JSONDecode(game:HttpGet(url))
			end)
			if not ok or type(result) ~= "table" or type(result.data) ~= "table" then
				Notify("Server Hop", "Could not fetch the server list", 4)
				hopping = false
				return
			end
			local candidates = {}
			for _, s in ipairs(result.data) do
				if s.id ~= game.JobId and s.playing and s.maxPlayers and s.playing < s.maxPlayers then
					table.insert(candidates, s)
				end
			end
			if #candidates == 0 then
				Notify("Server Hop", "No available servers found", 4)
				hopping = false
				return
			end
			local pick = candidates[math.random(#candidates)]
			local ok2, err = pcall(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, pick.id, LocalPlayer) end)
			if not ok2 then Notify("Server Hop", "Failed: " .. tostring(err), 4) end
			task.delay(5, function() hopping = false end)
		end)
	end

	local info = AddSection(server, "Server Info", "Main", "Server")
	Refs.srvPlayers = AddInfo(info, "Players", #Players:GetPlayers() .. "/" .. Players.MaxPlayers)
	AddInfo(info, "Place ID", tostring(game.PlaceId))
	AddInfo(info, "Job ID", game.JobId ~= "" and game.JobId or "Studio / Private")
	Refs.srvAge = AddInfo(info, "Server age", "--")
	local actions = AddSection(server, "Actions", "Main", "Server")
	AddButton(actions, "Copy Job ID", function() copyText(game.JobId, "Job ID") end)
	AddButton(actions, "Copy Place ID", function() copyText(tostring(game.PlaceId), "Place ID") end)
	AddButton(actions, "Rejoin Server", rejoin)
	AddButton(actions, "Server Hop", serverHop)
end

-- ---------- 10.4 SETTINGS ----------
local UnloadHub -- definido na seção 13
do
	local scroll = createTab("Settings", ICONS.Settings, true, ORDER.Settings)
	local ui = AddSection(scroll, "Interface", "Settings")

	if not IS_TOUCH then
		local row = create("Frame", {Size = UDim2.new(1, 0, 0, 38), BackgroundTransparency = 1,
			LayoutOrder = nextOrder(ui.Frame), Parent = ui.Frame})
		label({Size = UDim2.new(1, -130, 1, 0), Position = UDim2.new(0, 14, 0, 0), Text = "Menu Keybind", Parent = row})
		Refs.keyBtn = create("TextButton", {
			Size = UDim2.fromOffset(96, 26), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
			BackgroundColor3 = COLORS.CardHeader, Text = toggleKey.Name, TextColor3 = COLORS.TextMain, TextSize = 11,
			Font = Enum.Font.GothamBold, AutoButtonColor = false, Parent = row,
		})
		corner(Refs.keyBtn, 6)
		stroke(Refs.keyBtn, COLORS.Stroke, 1)
		Refs.keyBtn.MouseButton1Click:Connect(function()
			listeningKey = true
			Sfx.Play("Click")
			Refs.keyBtn.Text = "Press a key..."
			tween(Refs.keyBtn, 0.15, {BackgroundColor3 = COLORS.AccentDark})
		end)
		registerRow(ui, row, "Menu Keybind")
	end

	AddSlider(ui, "UI Scale", 0.6, 1.4, 1, 0.05, "x", function(v)
		userScale = v
		currentScale = computeScale()
		HubScale.Scale = currentScale
		hubCenter = clampCenter(hubCenter)
		MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
	end)
	AddToggle(ui, "Notifications", "Show pop-up messages", true, function(on) notificationsEnabled = on end)
	AddToggle(ui, "Snow Effect", "Disable for better performance", State.Snow, function(on)
		State.Snow = on
		SnowContainer.Visible = on and not Perf.Lite and isOpen
	end)

	-- Efeitos sonoros com controle de volume
	local snd = AddSection(scroll, "Sound", "Settings")
	AddToggle(snd, "Sound Effects", "Clicks, toggles, menu and alerts", Sfx.Enabled, function(on)
		Sfx.Enabled = on
	end)
	AddSlider(snd, "Volume", 0, 100, math.floor(Sfx.Volume * 100), 5, "%", function(v)
		Sfx.Volume = v / 100
	end)
	AddButton(snd, "Test Sound", function()
		Sfx.Play("On")
		task.delay(0.18, function() Sfx.Play("Notify") end)
	end)

	local danger = AddSection(scroll, "Danger Zone", "Settings")
	AddButton(danger, "Unload Hub", function() UnloadHub() end, true)
end

-- ================================================================= --
-- 11. SISTEMA DE SCRIPTS POR JOGO
--  Cada jogo vira uma ABA própria e SÓ aparece dentro do jogo dele.
--  Para adicionar outro jogo: copie o bloco RegisterGame({...}) e
--  preencha, em ordem de confiança:
--    PlaceIds / GameIds  -> detecção EXATA (não depende de idioma)
--    Names               -> nomes do jogo (ignora maiúsculas, acentos e símbolos)
--    Detect              -> função opcional que olha o conteúdo do jogo
--  O script só é carregado se UM desses métodos reconhecer o jogo.
-- ================================================================= --
local cleanupFns = {}
local GameModules = {}
local loadedGames = {}

local function RegisterGame(def)
	table.insert(GameModules, def)
end

local function buildGameContext(def, group)
	return {
		Page = function(name, icon)
			local scroll = group.Add(name, icon or ICONS.Merge)
			return {Section = function(title) return AddSection(scroll, title, def.TabName, name) end}
		end,
		Notify = Notify, ScreenGui = ScreenGui, Icons = ICONS, copy = clipboardFn,
		track = track,
		IsVisible = function() return isOpen and tabs[def.TabName] ~= nil and tabs[def.TabName].Active end,
		SetAntiAfk = function(on) setAntiAfk(on); PerfUI.afk.Set(on, true) end,
		SetLowGfx = function(on) Perf.autoFB = false; setLowGfx(on); PerfUI.gfx.Set(on, true) end,
		SetNoRender = function(on) setNoRender(on); PerfUI.noRender.Set(on, true) end,
		OnUnload = function(fn) table.insert(cleanupFns, fn) end,
	}
end

-- Deixa só letras/números minúsculos: "[UPDATE] Merge a Mini Army!" -> "updatemergeaminiarmy"
local function normalize(text)
	return (string.gsub(string.lower(tostring(text or "")), "[^%w]", ""))
end

local function matchesGame(def, gameName)
	for _, id in ipairs(def.PlaceIds or {}) do
		if id == game.PlaceId then return true end
	end
	for _, id in ipairs(def.GameIds or {}) do
		if id == game.GameId then return true end
	end
	local candidates = {}
	if def.NameMatch then candidates[#candidates + 1] = def.NameMatch end
	for _, n in ipairs(def.Names or {}) do candidates[#candidates + 1] = n end
	if #candidates > 0 then
		local haystacks = {normalize(gameName), normalize(game.Name)}
		for _, c in ipairs(candidates) do
			local nc = normalize(c)
			if #nc > 0 then
				for _, h in ipairs(haystacks) do
					if #h > 0 and string.find(h, nc, 1, true) then return true end
				end
			end
		end
	end
	return false
end

-- Só é chamado quando o jogo atual combina com o script (a aba nem existe nos outros jogos)
local function loadGame(def)
	if loadedGames[def.Name] or not ScreenGui.Parent then return false end
	loadedGames[def.Name] = true
	local page = createTab(def.TabName, def.Icon or ICONS.Merge, false, ORDER.Game)
	local group = createSubGroup(def.TabName, page)
	local ok, err = pcall(def.Build, buildGameContext(def, group))
	if ok then
		Refs.gameScript.Text = def.Name
		Notify("Game Script", def.Name .. " loaded", 3)
	else
		warn("[ShadowHub] " .. def.Name .. ": " .. tostring(err))
		Refs.gameScript.Text = def.Name .. " (error)"
		Notify("Game Script", "Failed to load " .. def.Name, 4)
	end
	return ok
end

-- ================================================================= --
-- 12. JOGO: MERGE A MINI ARMY
--  Sub-abas: Quick | Merge | Economy | Combat | ESP | Tools
--
--  AUTO MERGE: o SEU personagem anda sozinho até os pares iguais da
--  SUA base (caminhada normal, SEM teleporte). Ao passar por cima, o
--  jogo junta. Se você andar com teclado/joystick, o auto pausa e
--  retoma quando você parar. Ordem: remote aprendido > andar.
--  Teleporte só existe no Auto Base Attack (opcional, como antes).
--
--  AUTO REBIRTH / UPGRADE: modo silencioso. Clica nos botões por trás
--  da interface: nada de pop-up, notificação ou som no seu lado.
-- ================================================================= --
RegisterGame({
	Name = "Merge a Mini Army",
	TabName = "Mini Army",
	Icon = ICONS.Merge,
	PlaceIds = {},                       -- opcional: PlaceId exato (veja em Home > Copy Game Info)
	GameIds = {},                        -- opcional: GameId exato
	Names = {"Merge a Mini Army", "Merge Mini Army", "Mini Army"},
	-- Plano B (independe de idioma): olha os NOMES de remotes/objetos do jogo, que o
	-- desenvolvedor escreve em inglês. Exige "merge" + algo de exército juntos.
	Detect = function()
		local hasMerge, hasArmy = false, false
		local ARMY = {"army", "deploy", "airdrop", "garrison", "outpost", "troop", "soldier"}
		local function check(inst)
			local n = string.lower(inst.Name)
			if not hasMerge and string.find(n, "merge", 1, true) then hasMerge = true end
			if not hasArmy and hasAny(n, ARMY) then hasArmy = true end
		end
		local i = 0
		for _, d in ipairs(game:GetService("ReplicatedStorage"):GetDescendants()) do
			check(d)
			i = i + 1
			if hasMerge and hasArmy then return true end
			if i % 800 == 0 then task.wait() end
			if i > 8000 then break end
		end
		for _, c in ipairs(workspace:GetChildren()) do
			check(c)
			for _, g in ipairs(c:GetChildren()) do
				check(g)
				i = i + 1
				if i % 800 == 0 then task.wait() end
			end
			if hasMerge and hasArmy then return true end
		end
		return hasMerge and hasArmy
	end,
	Build = function(ctx)
		local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
		local running = true
		local stats = {clicks = 0, rebirths = 0, merges = 0, fails = 0, captures = 0}
		local ctl = {}
		local MERGE_WORDS = {"merge", "combine", "fuse"}

		local cfg = {
			-- merge (anda de verdade até os pares; sem teleporte)
			Merge = false, MergeDelay = 0.1, UseRemote = true, AutoLearn = true,
			WalkTimeout = 10,        -- segundos máximos para chegar em cada unidade
			Drag = false, MergeBtn = false, UnitWords = {}, BaseRadius = 70,
			-- economia
			Upgrade = false, UpgradeEvery = 1, UpgradeWords = {}, Rebirth = false, RebirthEvery = 20,
			Collect = false, Spin = false, Equip = false, Buy = false, ActionDelay = 0.8,
			Silent = true,           -- clica sem precisar abrir menus
			MouseFallback = false,   -- clique de mouse virtual (só com botão visível)
			-- combate
			Deploy = false, DeployEvery = 5, Battle = false, Attack = false, AtkEvent = true, AtkCamp = true,
			AtkBase = false, AtkTeleport = true, KeepDeployed = true, AtkTimeout = 12,
			AutoCapture = false, Range = 25, Instant = false,
			-- tools
			Custom = false, CustomWords = {}, AfkRebirth = false,
		}

		-- Auto Merge ligado por botão rápido OU por "Merge Now" (uma rodada completa)
		local mergeOnce = false

		local function isPlayerChar(i) return i:IsA("Model") and Players:GetPlayerFromCharacter(i) ~= nil end
		local function posOf(i)
			if i:IsA("BasePart") then return i.Position end
			if i:IsA("Model") then return i:GetPivot().Position end
			return nil
		end

		-- ======================================================
		-- A) MINHA BASE (o merge só atua dentro dela)
		-- ======================================================
		local base = {model = nil, center = nil, radius = 70, label = "Not set"}
		local lastBaseWarn = 0

		local function matchesMe(v)
			return v == LocalPlayer or v == LocalPlayer.UserId or v == LocalPlayer.Name
				or v == LocalPlayer.DisplayName or tostring(v) == tostring(LocalPlayer.UserId)
		end

		local function ownedByMe(inst)
			for k, v in pairs(inst:GetAttributes()) do
				if hasAny(string.lower(k), {"owner", "player", "user"}) and matchesMe(v) then return true end
			end
			for _, c in ipairs(inst:GetChildren()) do
				if c:IsA("ObjectValue") then
					if c.Value == LocalPlayer then return true end
				elseif c:IsA("StringValue") or c:IsA("IntValue") or c:IsA("NumberValue") then
					if hasAny(string.lower(c.Name), {"owner", "player", "user"}) and matchesMe(c.Value) then return true end
				end
			end
			local ln = string.lower(inst.Name)
			if string.find(ln, string.lower(LocalPlayer.Name), 1, true) then return true end
			if #LocalPlayer.DisplayName >= 4 and string.find(ln, string.lower(LocalPlayer.DisplayName), 1, true) then return true end
			return false
		end

		local function regionOf(inst)
			if inst:IsA("Model") then
				local cf, size = inst:GetBoundingBox()
				return cf.Position, math.clamp(math.max(size.X, size.Z) / 2 + 8, 20, 400)
			end
			local minV, maxV, n
			for _, d in ipairs(inst:GetDescendants()) do
				if d:IsA("BasePart") then
					local p = d.Position
					minV = minV and Vector3.new(math.min(minV.X, p.X), math.min(minV.Y, p.Y), math.min(minV.Z, p.Z)) or p
					maxV = maxV and Vector3.new(math.max(maxV.X, p.X), math.max(maxV.Y, p.Y), math.max(maxV.Z, p.Z)) or p
					n = (n or 0) + 1
					if n > 600 then break end
				end
			end
			if not minV then return nil end
			return (minV + maxV) / 2, math.clamp((maxV - minV).Magnitude / 2 + 8, 20, 400)
		end

		-- Procura automaticamente o plot/base que pertence ao seu jogador
		local function detectBase()
			local found
			local count = 0
			local function try(inst)
				if found then return end
				if (inst:IsA("Model") or inst:IsA("Folder")) and inst ~= LocalPlayer.Character
					and not isPlayerChar(inst) and ownedByMe(inst) then
					found = inst
				end
			end
			for _, c in ipairs(workspace:GetChildren()) do
				if found then break end
				try(c)
				if not found and not c:IsA("Terrain") and not c:IsA("Camera") and not isPlayerChar(c) then
					for _, g in ipairs(c:GetChildren()) do
						try(g)
						count = count + 1
						if found or count > 3000 then break end
					end
				end
			end
			if found then
				local ok, center, radius = pcall(regionOf, found)
				if ok and center then
					base.model, base.center, base.radius, base.label = found, center, radius, found.Name
					return true
				end
			end
			return false
		end

		local function setBaseHere()
			local root = getRoot()
			if not root then return false end
			base.model, base.center, base.radius = nil, root.Position, cfg.BaseRadius
			base.label = "Manual (" .. math.floor(root.Position.X) .. ", " .. math.floor(root.Position.Z) .. ")"
			return true
		end

		local function ensureBase(quiet)
			if base.center then return true end
			if detectBase() then return true end
			if not quiet and os.clock() - lastBaseWarn > 10 then
				lastBaseWarn = os.clock()
				ctx.Notify("Base not found", "Stand in your base and press Set Base Here", 5)
			end
			return false
		end

		local function inMyBase(inst)
			if not base.center then return false end
			if base.model and base.model.Parent and inst:IsDescendantOf(base.model) then return true end
			local p = posOf(inst)
			return p ~= nil and (p - base.center).Magnitude <= base.radius
		end

		-- ======================================================
		-- B) UNIDADES, GRUPOS E PARES IGUAIS
		-- ======================================================
		local META = {level = true, lvl = true, tier = true, rank = true, grade = true, stage = true,
			mergelevel = true, unitlevel = true, evolution = true, star = true, stars = true,
			unittype = true, type = true, class = true, kind = true, unitname = true, rarity = true}
		local sizeCache = setmetatable({}, {__mode = "k"})
		local cooldown = setmetatable({}, {__mode = "k"})
		local failCount = setmetatable({}, {__mode = "k"})
		local overlap = OverlapParams.new()
		overlap.FilterType = Enum.RaycastFilterType.Exclude
		overlap.MaxParts = 2500
		local scan = {t = 0, groups = {}, units = 0, pairs = 0}

		local function modelSize(m)
			local s = sizeCache[m]
			if s then return s end
			local ok, ext = pcall(function() return m:GetExtentsSize() end)
			s = ok and math.max(ext.X, ext.Y, ext.Z) or 99
			sizeCache[m] = s
			return s
		end

		local function topSmallModel(p)
			local best
			local cur = p.Parent
			local depth = 0
			while cur and cur ~= workspace and depth < 6 do
				if cur:IsA("Model") then
					if modelSize(cur) < 16 then best = cur else break end
				end
				cur = cur.Parent
				depth = depth + 1
			end
			return best
		end

		local function unitPart(u)
			if u:IsA("BasePart") then return u end
			return u.PrimaryPart or u:FindFirstChild("HumanoidRootPart") or u:FindFirstChildWhichIsA("BasePart", true)
		end

		-- Chave que identifica "unidades iguais" (mesmo tipo + mesmo nível)
		local function unitKey(m)
			local parts, meta = {}, false
			for k, v in pairs(m:GetAttributes()) do
				local lk = string.lower(k)
				local tv = type(v)
				if META[lk] and (tv == "number" or tv == "string" or tv == "boolean") then
					parts[#parts + 1] = lk .. ":" .. tostring(v)
					meta = true
				end
			end
			for _, c in ipairs(m:GetChildren()) do
				if c:IsA("ValueBase") and META[string.lower(c.Name)] then
					local ok, v = pcall(function() return c.Value end)
					if ok and v ~= nil then
						parts[#parts + 1] = string.lower(c.Name) .. ":" .. tostring(v)
						meta = true
					end
				end
			end
			local name = m.Name
			if meta then
				name = string.gsub(name, "%d+", "")
			else
				name = string.gsub(name, "%x%x%x%x%x%x%x%x%-%x%x%x%x.*$", "")
				if not string.find(name, "%d") then
					for _, d in ipairs(m:GetDescendants()) do
						if d:IsA("TextLabel") and string.find(d.Text, "%d") then
							parts[#parts + 1] = "t:" .. d.Text
							break
						end
					end
				end
			end
			local unitLike = meta
			if not unitLike then
				unitLike = (m:FindFirstChildOfClass("Humanoid") ~= nil) or (m:FindFirstChildOfClass("AnimationController") ~= nil)
			end
			if #cfg.UnitWords > 0 then unitLike = hasAny(string.lower(m.Name), cfg.UnitWords) end
			if not unitLike then return nil end
			table.sort(parts)
			return name .. "|" .. table.concat(parts, ",")
		end

		local function scanUnits(force)
			local now = os.clock()
			if not force and now - scan.t < (Perf.Lite and 2 or 0.6) then return scan end
			scan.t = now
			local groups, total, seen = {}, 0, {}
			local function consider(m)
				if seen[m] or not m.Parent then return end
				seen[m] = true
				if isPlayerChar(m) then return end
				if not inMyBase(m) then return end
				local key = unitKey(m)
				if key then
					local g = groups[key]
					if not g then g = {}; groups[key] = g end
					g[#g + 1] = m
					total = total + 1
				end
			end
			if base.center then
				overlap.FilterDescendantsInstances = {LocalPlayer.Character, ESPFolder}
				local ok, found = pcall(function() return workspace:GetPartBoundsInRadius(base.center, base.radius, overlap) end)
				if ok and found then
					for i = 1, #found do
						local part = found[i]
						local m = topSmallModel(part)
						if not m and #cfg.UnitWords > 0 and hasAny(string.lower(part.Name), cfg.UnitWords) then m = part end
						if m then consider(m) end
					end
				end
			end
			local pairsN = 0
			for _, g in pairs(groups) do pairsN = pairsN + math.floor(#g / 2) end
			scan.groups, scan.units, scan.pairs = groups, total, pairsN
			return scan
		end

		-- ======================================================
		-- C) EXECUÇÃO DO MERGE (anda de verdade, SEM teleporte)
		--    1) remote aprendido (se existir, não precisa andar)
		--    2) andar até a unidade A, depois até a B
		-- ======================================================
		local learned = env.__ShadowMiniLearn
		local recording, captured = false, nil

		-- Prompts de merge de cada unidade (em cache, para não varrer descendentes toda hora)
		local promptCache = setmetatable({}, {__mode = "k"})
		local function mergePrompts(u)
			local list = promptCache[u]
			if list then return list end
			list = {}
			for _, d in ipairs(u:GetDescendants()) do
				if d:IsA("ProximityPrompt")
					and hasAny(string.lower(d.ActionText .. " " .. d.ObjectText .. " " .. d.Name), MERGE_WORDS) then
					list[#list + 1] = d
				end
			end
			promptCache[u] = list
			return list
		end

		local function firePrompts(u)
			if not fireproximityprompt then return end
			for _, d in ipairs(mergePrompts(u)) do
				if d.Parent and d.Enabled then pcall(fireproximityprompt, d) end
			end
		end

		-- Toca a unidade (sem mover o personagem): complemento ao andar
		local function touchUnit(u)
			local p, root = unitPart(u), getRoot()
			if not (p and root) then return end
			if firetouchinterest then
				pcall(function()
					firetouchinterest(root, p, 0)
					firetouchinterest(root, p, 1)
				end)
			end
			firePrompts(u)
		end

		local function fireLearned(a, b)
			local L = learned
			if not (L and L.remote and L.remote.Parent) then return false end
			local args = {}
			for i = 1, L.args.n do args[i] = L.args[i] end
			if a and #L.inst >= 2 then
				args[L.inst[1]] = a
				args[L.inst[2]] = b
			elseif a and #L.inst == 1 then
				args[L.inst[1]] = a
			end
			task.spawn(function()
				pcall(function()
					if L.method == "InvokeServer" then L.remote:InvokeServer(table.unpack(args, 1, L.args.n))
					else L.remote:FireServer(table.unpack(args, 1, L.args.n)) end
				end)
			end)
			return true
		end

		-- Detecta se VOCÊ está controlando o personagem (teclado / joystick / setas)
		local MOVE_KEYS = {Enum.KeyCode.W, Enum.KeyCode.A, Enum.KeyCode.S, Enum.KeyCode.D,
			Enum.KeyCode.Up, Enum.KeyCode.Down, Enum.KeyCode.Left, Enum.KeyCode.Right}
		local function userIsMoving()
			if Mv.controls then
				local ok, v = pcall(function() return Mv.controls:GetMoveVector() end)
				if ok and v and v.Magnitude > 0.1 then return true end
			end
			for _, k in ipairs(MOVE_KEYS) do
				if UserInputService:IsKeyDown(k) then return true end
			end
			return false
		end

		local function mergeActive()
			return running and ctx.ScreenGui.Parent ~= nil and (cfg.Merge or mergeOnce)
		end

		local function stopWalk()
			local hum, root = getHum(), getRoot()
			if hum and root then pcall(function() hum:MoveTo(root.Position) end) end
		end

		-- Anda até uma posição com o próprio personagem (caminhada normal, sem CFrame/teleporte)
		-- Retorna: "arrived", "timeout", "user" (você assumiu), "stop" (desligou / morreu)
		local function walkTo(pos, timeout)
			local root, hum = getRoot(), getHum()
			if not (root and hum) or State.Fly then return "stop" end
			hum:MoveTo(pos)
			local t0 = os.clock()
			local last, stuckAt = root.Position, os.clock()
			while mergeActive() do
				if userIsMoving() then
					stopWalk()
					return "user"
				end
				root, hum = getRoot(), getHum()
				if not (root and hum) then return "stop" end
				local cur = root.Position
				local flat = Vector3.new(cur.X - pos.X, 0, cur.Z - pos.Z).Magnitude
				if flat <= 3.5 then
					stopWalk()
					return "arrived"
				end
				if os.clock() - t0 > timeout then
					stopWalk()
					return "timeout"
				end
				if (cur - last).Magnitude > 0.35 then
					last, stuckAt = cur, os.clock()
				elseif os.clock() - stuckAt > 0.9 then
					-- travou em algo: pula e tenta de novo
					hum.Jump = true
					hum:MoveTo(pos)
					stuckAt = os.clock()
				end
				task.wait(0.1)
			end
			stopWalk()
			return "stop"
		end

		-- Par mais próximo de você (pares disponíveis, sem cooldown)
		local function nearestPair()
			local sc = scanUnits()
			local root = getRoot()
			local now = os.clock()
			local bestA, bestB, bestScore
			for _, list in pairs(sc.groups) do
				local n = #list
				for i = 1, n do
					local a = list[i]
					local pa = unitPart(a)
					if pa and a.Parent and (not cooldown[a] or cooldown[a] < now) then
						for j = i + 1, n do
							local b = list[j]
							local pb = unitPart(b)
							if pb and b.Parent and (not cooldown[b] or cooldown[b] < now) then
								local score = (pa.Position - pb.Position).Magnitude
								if root then score = score + (pa.Position - root.Position).Magnitude * 0.5 end
								if not bestScore or score < bestScore then
									bestA, bestB, bestScore = a, b, score
								end
							end
						end
					end
				end
			end
			if bestA then return {bestA, bestB} end
			return nil
		end

		local function isMerged(a, b, keyA, keyB)
			return (not a.Parent) or (not b.Parent) or unitKey(a) ~= keyA or unitKey(b) ~= keyB
		end

		local function markOk(a, b)
			stats.merges = stats.merges + 1
			failCount[a], failCount[b] = nil, nil
		end

		local function markFail(a, b)
			local n = (failCount[a] or 0) + 1
			failCount[a], failCount[b] = n, n
			local t = os.clock() + math.min(2 * n, 12)
			cooldown[a], cooldown[b] = t, t
			stats.fails = stats.fails + 1
		end

		local function mergeOnePair(a, b)
			local pa, pb = unitPart(a), unitPart(b)
			if not (pa and pb) then return end
			local keyA, keyB = unitKey(a), unitKey(b)

			if cfg.Drag then pcall(function() a:PivotTo(CFrame.new(pb.Position)) end) end

			-- 1) remote aprendido (sem andar)
			if cfg.UseRemote and learned and learned.remote and learned.remote.Parent and not learned.direct then
				fireLearned(a, b)
				task.wait(0.4)
				if isMerged(a, b, keyA, keyB) then
					markOk(a, b)
					return
				end
			end

			-- 2) anda até a primeira unidade, depois até a segunda
			local status = walkTo(pa.Position, cfg.WalkTimeout)
			if status == "arrived" then
				touchUnit(a)
				task.wait(0.3)
				status = walkTo(pb.Position, cfg.WalkTimeout)
				if status == "arrived" then
					touchUnit(b)
					task.wait(0.5)
				end
			end

			if status == "arrived" then
				if isMerged(a, b, keyA, keyB) then markOk(a, b) else markFail(a, b) end
			elseif status == "timeout" then
				markFail(a, b)
			end
			-- "user" / "stop": não conta como falha, só pausa
		end

		-- Loop do Auto Merge
		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				if cfg.Merge or mergeOnce then
					if ensureBase() then
						if cfg.UseRemote and learned and learned.direct and learned.remote then
							local sc = scanUnits()
							if sc.pairs > 0 then
								fireLearned()
								stats.merges = stats.merges + 1
							else
								mergeOnce = false
							end
							task.wait(math.max(cfg.MergeDelay, Perf.Lite and 0.4 or 0.1))
						elseif userIsMoving() then
							-- Você está andando: o personagem fica livre (merges acontecem ao passar por cima)
							task.wait(0.2)
						else
							local pair = nearestPair()
							if pair then
								local ok, err = pcall(mergeOnePair, pair[1], pair[2])
								if not ok then warn("[ShadowHub] merge: " .. tostring(err)) end
								scan.t = 0
								task.wait(math.max(cfg.MergeDelay, Perf.Lite and 0.3 or 0.05))
							else
								mergeOnce = false
								task.wait(0.4)
							end
						end
					else
						mergeOnce = false
						task.wait(1)
					end
				else
					task.wait(0.3)
				end
			end
		end)

		-- Aprender o remote do merge (você faz 1 merge à mão e o script copia)
		local SKIP_REMOTE = {"position", "move", "camera", "ping", "heartbeat", "analytics", "log", "cursor", "mouse", "look"}
		local function onRemote(remote, method, ...)
			if not recording then return end
			local lname = string.lower(remote.Name)
			if hasAny(lname, SKIP_REMOTE) then return end
			local args = table.pack(...)
			local inst = {}
			for i = 1, args.n do
				local v = args[i]
				if typeof(v) == "Instance" and (v:IsA("Model") or v:IsA("BasePart")) and inMyBase(v) then
					inst[#inst + 1] = i
				end
			end
			local byName = hasAny(lname, MERGE_WORDS)
			-- precisa ter nome de merge OU duas unidades da sua base como argumento
			if byName or #inst >= 2 then
				if not captured or byName then
					captured = {remote = remote, method = method, args = args, inst = inst,
						byName = byName, direct = (#inst == 0)}
				end
			end
		end

		local function startLearning(seconds, quiet)
			if recording then return end
			if not installHook() then
				if not quiet then ctx.Notify("Learn", "Executor lacks hookmetamethod. Walk mode still works", 5) end
				return
			end
			if not ensureBase(quiet) then return end
			captured, recording = nil, true
			env.__ShadowHookFn = onRemote
			if not quiet then ctx.Notify("Learning", "Merge two units BY HAND now (" .. seconds .. "s)", 6) end
			task.spawn(function()
				local t0 = os.clock()
				while running and recording and not captured and os.clock() - t0 < seconds do task.wait(0.25) end
				recording = false
				env.__ShadowHookFn = nil
				if captured then
					learned = captured
					env.__ShadowMiniLearn = learned
					if Refs.mRemote then Refs.mRemote.Text = learned.remote.Name end
					ctx.Notify("Learned", learned.remote.Name .. (learned.direct and " (direct)" or " (targets)"), 4)
				elseif not quiet then
					ctx.Notify("Learn", "Nothing captured. Try again", 4)
				end
			end)
		end

		-- ======================================================
		-- D) BOTÕES DA INTERFACE DO JOGO (Economy / Combat / extras)
		-- ======================================================
		local CATS = {
			{key = "Rebirth",  words = {"rebirth"}, boostOk = true},
			{key = "Upgrade",  words = {"upgrade"}},
			{key = "Deploy",   words = {"deploy"}},
			{key = "Reset",    words = {"reset army", "recall", "retreat"}},
			{key = "MergeBtn", words = {"merge", "combine"}, boostOk = true},
			{key = "Equip",    words = {"equip best", "equip all", "best units"}},
			{key = "Collect",  words = {"collect", "claim"}},
			{key = "Spin",     words = {"spin"}},
			{key = "Battle",   words = {"start battle", "next wave", "next stage", "start wave"}},
			{key = "Buy",      words = {"buy", "spawn", "summon", "recruit", "hire"}},
		}
		local BLOCK_HARD = {"robux", "r$", "gamepass", "game pass", "premium", "gift", "donat", "purchase", "vip"}
		local BOOST = {"x2", "2x", "x3", "3x", "x4", "4x"}
		local VIM
		pcall(function() VIM = game:GetService("VirtualInputManager") end)

		local buttons, trackingButtons, scanDone = {}, false, false
		local function addButton(d)
			if (d:IsA("TextButton") or d:IsA("ImageButton")) and not d:IsDescendantOf(ctx.ScreenGui) then
				buttons[d] = {text = nil, t = 0}
			end
		end
		local function ensureButtonTracking()
			if trackingButtons then return end
			trackingButtons = true
			task.spawn(function()
				local i = 0
				for _, d in ipairs(PlayerGui:GetDescendants()) do
					addButton(d)
					i = i + 1
					if i % 300 == 0 then task.wait() end
				end
				scanDone = true
			end)
			ctx.track(PlayerGui.DescendantAdded:Connect(addButton))
			ctx.track(PlayerGui.DescendantRemoving:Connect(function(d) buttons[d] = nil end))
		end

		local function refreshInfo(b, info, now)
			if info.text and now - info.t < 2 then return end
			local t = b:IsA("TextButton") and b.Text or ""
			for _, c in ipairs(b:GetDescendants()) do
				if c:IsA("TextLabel") or c:IsA("TextButton") then t = t .. " " .. c.Text end
			end
			t = string.lower(t .. " " .. b.Name)
			info.text, info.t = t, now
			info.block = hasAny(t, BLOCK_HARD)
			info.boost = hasAny(t, BOOST)
			-- Confirmação: só o TEXTO do próprio botão (evita clicar em "eyes", "yesterday" etc.)
			local own = string.lower(b:IsA("TextButton") and b.Text or "")
			info.confirm = (own == "yes" or own == "confirm" or own == "ok" or string.find(own, "confirm", 1, true) ~= nil)
			info.cat = nil
			for _, cat in ipairs(CATS) do
				if hasAny(t, cat.words) then info.cat = cat break end
			end
		end

		local function isShown(b)
			if not b.Parent then return false end
			local cur = b
			while cur and cur ~= PlayerGui do
				if cur:IsA("GuiObject") and not cur.Visible then return false end
				if cur:IsA("ScreenGui") and not cur.Enabled then return false end
				cur = cur.Parent
			end
			return b.AbsoluteSize.X > 0 and b.AbsoluteSize.Y > 0
		end

		local function fireSignal(sig)
			if getconnections then
				local ok, conns = pcall(getconnections, sig)
				if ok and conns and #conns > 0 then
					for _, c in ipairs(conns) do pcall(function() c:Fire() end) end
					return true
				end
			end
			return false
		end

		-- Clique SILENCIOSO: dispara os eventos do botão direto (funciona com o menu fechado e
		-- sem mexer no mouse). O clique virtual só entra se você ativar "Mouse fallback".
		local function clickButton(b)
			if fireSignal(b.MouseButton1Click) then return true end
			if fireSignal(b.Activated) then return true end
			if firesignal then
				if pcall(firesignal, b.MouseButton1Click) then return true end
				if pcall(firesignal, b.Activated) then return true end
			end
			if VIM and cfg.MouseFallback and isShown(b) then
				local sg = b:FindFirstAncestorWhichIsA("ScreenGui")
				local inset = (sg and sg.IgnoreGuiInset) and Vector2.zero or GuiService:GetGuiInset()
				local pos = b.AbsolutePosition + b.AbsoluteSize / 2 + inset
				return pcall(function()
					VIM:SendMouseButtonEvent(pos.X, pos.Y, 0, true, game, 0)
					VIM:SendMouseButtonEvent(pos.X, pos.Y, 0, false, game, 0)
				end)
			end
			return false
		end

		local lastConfirm = 0
		local function clickCategory(due, now, allowConfirm)
			local list = {}
			for b in pairs(buttons) do list[#list + 1] = b end
			local n = 0
			for i = 1, #list do
				local b = list[i]
				local info = buttons[b]
				if info and b.Parent then
					refreshInfo(b, info, now)
					if not info.block then
						local k1 = info.cat and info.cat.key
						local key
						if k1 and due[k1] and (not info.boost or info.cat.boostOk) then
							key = k1
						elseif due.Upgrade and #cfg.UpgradeWords > 0 and hasAny(info.text, cfg.UpgradeWords) then
							key = "Upgrade"
						elseif due.Custom and #cfg.CustomWords > 0 and hasAny(info.text, cfg.CustomWords) then
							key = "Custom"
						elseif allowConfirm and now < lastConfirm and info.confirm then
							key = "Confirm"
						end
						-- Filtro de upgrades escolhidos
						if key == "Upgrade" and #cfg.UpgradeWords > 0 and not hasAny(info.text, cfg.UpgradeWords) then key = nil end
						-- Modo silencioso: não exige o botão visível
						if key and (cfg.Silent or isShown(b)) then
							if clickButton(b) then
								n = n + 1
								stats.clicks = stats.clicks + 1
								if key == "Rebirth" then
									stats.rebirths = stats.rebirths + 1
									lastConfirm = now + 4
								end
							end
						end
					end
				end
				if i % 60 == 0 then task.wait() end
			end
			return n
		end

		-- Categorias que rodam em silêncio total (sem nenhuma mensagem na tela)
		local QUIET_KEYS = {Rebirth = true, Upgrade = true}

		local function clickNow(key)
			ensureButtonTracking()
			local waited = 0
			while not scanDone and waited < 3 do task.wait(0.1); waited = waited + 0.1 end
			local n = clickCategory({[key] = true}, os.clock(), key == "Rebirth")
			if QUIET_KEYS[key] then return end
			ctx.Notify(key, n > 0 and (n .. " click(s)") or (cfg.Silent and "No matching button found" or "No visible button. Open the menu in the game"), 3)
		end

		local DUE_KEYS = {"Rebirth", "Upgrade", "Deploy", "MergeBtn", "Collect", "Spin", "Equip", "Battle", "Buy", "Custom"}
		local function everyOf(k)
			local v
			if k == "Rebirth" then v = cfg.RebirthEvery
			elseif k == "Upgrade" then v = cfg.UpgradeEvery
			elseif k == "Deploy" then v = cfg.DeployEvery
			else v = cfg.ActionDelay end
			return Perf.Lite and math.max(v, 1) or v
		end

		task.spawn(function()
			local last = {}
			while running and ctx.ScreenGui.Parent do
				task.wait(0.25)
				local now = os.clock()
				local due, any = {}, false
				for _, k in ipairs(DUE_KEYS) do
					if cfg[k] and now - (last[k] or 0) >= everyOf(k) then
						due[k] = true
						last[k] = now
						any = true
					end
				end
				if any then
					ensureButtonTracking()
					pcall(clickCategory, due, now, cfg.Rebirth)
				end
			end
		end)

		-- ======================================================
		-- E) TERRITÓRIOS, EVENTOS E BASE ATTACK
		-- ======================================================
		local KEYWORDS = {"captur", "conquer", "garrison", "compound", "airbase", "territor", "airdrop", "camp", "outpost", "crate", "event"}
		local prompts, originalHold = {}, {}
		local territoryCache = setmetatable({}, {__mode = "k"})
		local trackingPrompts = false
		local atkDone = setmetatable({}, {__mode = "k"})
		local atk = {status = "Idle", target = "None", event = "None"}

		local function promptPart(p)
			local par = p.Parent
			if not par then return nil end
			if par:IsA("BasePart") then return par end
			if par:IsA("Attachment") then return par.Parent:IsA("BasePart") and par.Parent or nil end
			if par:IsA("Model") then return par.PrimaryPart or par:FindFirstChildWhichIsA("BasePart", true) end
			return nil
		end

		local function promptText(p)
			local par = p.Parent
			return string.lower(p.ActionText .. " " .. p.ObjectText .. " " .. (par and par.Name or "") .. " " .. ((par and par.Parent) and par.Parent.Name or ""))
		end

		local function isTerritory(p)
			local c = territoryCache[p]
			if c ~= nil then return c end
			if not p.Parent then return false end
			local r = hasAny(promptText(p), KEYWORDS)
			territoryCache[p] = r
			return r
		end

		local function kindOf(text)
			if hasAny(text, {"airdrop", "event", "crate", "drop"}) then return "Event", COLORS.Gold end
			if hasAny(text, {"camp", "outpost", "fort"}) then return "Camp", Color3.fromRGB(255, 90, 90) end
			return "Base", COLORS.AccentGlow
		end

		local function applyInstant(p)
			if p.HoldDuration > 0 then
				originalHold[p] = originalHold[p] or p.HoldDuration
				p.HoldDuration = 0
			end
		end
		local function restoreInstant()
			for p, h in pairs(originalHold) do
				if p.Parent then p.HoldDuration = h end
			end
			originalHold = {}
		end
		local function addPrompt(d)
			if d:IsA("ProximityPrompt") then
				prompts[d] = true
				if cfg.Instant then applyInstant(d) end
			end
		end
		local function ensurePromptTracking()
			if trackingPrompts then return end
			trackingPrompts = true
			task.spawn(function()
				local i = 0
				for _, d in ipairs(workspace:GetDescendants()) do
					addPrompt(d)
					i = i + 1
					if i % 400 == 0 then task.wait() end
				end
			end)
			ctx.track(workspace.DescendantAdded:Connect(addPrompt))
			ctx.track(workspace.DescendantRemoving:Connect(function(d)
				if prompts[d] then prompts[d] = nil; originalHold[d] = nil; territoryCache[d] = nil end
			end))
		end

		local function pickTarget()
			local root = getRoot()
			if not root then return nil end
			local best, bestScore
			local now = os.clock()
			for p in pairs(prompts) do
				if p.Parent and p.Enabled and isTerritory(p) and (atkDone[p] or 0) < now then
					local part = promptPart(p)
					if part and not (base.center and inMyBase(part)) then
						local kind = kindOf(promptText(p))
						if (kind == "Event" and cfg.AtkEvent) or (kind == "Camp" and cfg.AtkCamp) or (kind == "Base" and cfg.AtkBase) then
							local score = (part.Position - root.Position).Magnitude * (kind == "Event" and 0.3 or 1)
							if not bestScore or score < bestScore then best, bestScore = p, score end
						end
					end
				end
			end
			return best
		end

		-- Auto Base Attack: vai até o alvo, segura até capturar, passa para o próximo
		task.spawn(function()
			local origin, lastDeploy = nil, 0
			while running and ctx.ScreenGui.Parent do
				task.wait(0.4)
				if cfg.Attack and fireproximityprompt then
					ensurePromptTracking()
					if cfg.KeepDeployed and os.clock() - lastDeploy > 6 then
						lastDeploy = os.clock()
						ensureButtonTracking()
						clickCategory({Deploy = true}, os.clock(), false)
					end
					local p = pickTarget()
					if p then
						local part = promptPart(p)
						local root = getRoot()
						atk.target = part and part.Parent and part.Parent.Name or "Target"
						atk.status = "Attacking"
						if part and root then
							if cfg.AtkTeleport then
								if not origin then origin = root.CFrame end
								root.CFrame = CFrame.new(part.Position + Vector3.new(0, 3, 0))
								root.AssemblyLinearVelocity = Vector3.zero
								task.wait(0.15)
							end
							local t0 = os.clock()
							repeat
								pcall(fireproximityprompt, p)
								task.wait(0.25)
							until not p.Parent or not p.Enabled or os.clock() - t0 > cfg.AtkTimeout or not cfg.Attack or not running
							atkDone[p] = os.clock() + 20
							stats.captures = stats.captures + 1
						else
							atkDone[p] = os.clock() + 20
						end
					else
						atk.status = "Searching"
						atk.target = "None"
						if origin then
							local r = getRoot()
							if r then r.CFrame = origin end
							origin = nil
						end
					end
				else
					atk.status = cfg.Attack and "No fireproximityprompt" or "Idle"
				end
			end
		end)

		-- Captura simples por proximidade (sem teleporte)
		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(0.5)
				if cfg.AutoCapture and fireproximityprompt then
					local root = getRoot()
					if root then
						local rpos = root.Position
						for p in pairs(prompts) do
							if p.Parent and p.Enabled and isTerritory(p) then
								local part = promptPart(p)
								if part and (part.Position - rpos).Magnitude <= cfg.Range then pcall(fireproximityprompt, p) end
							end
						end
					end
				end
			end
		end)

		-- ======================================================
		-- F) ESP DO JOGO (pares de merge + alvos/eventos)
		-- ======================================================
		local function hashStr(s)
			local h = 7
			for i = 1, #s do h = (h * 31 + string.byte(s, i)) % 100003 end
			return h
		end

		ESP.Register("MiniPairs", function()
			if not ensureBase(true) then return {} end
			local sc = scanUnits()
			local out = {}
			for key, list in pairs(sc.groups) do
				if #list >= 2 then
					local col = Color3.fromHSV((hashStr(key) % 360) / 360, 0.75, 1)
					for _, u in ipairs(list) do
						local p = unitPart(u)
						if p and #out < 40 then
							out[#out + 1] = {part = p, hl = u, text = "MERGE x" .. #list, color = col}
						end
					end
				end
			end
			return out
		end)

		local nameCache = {t = 0, list = {}}
		local NAME_WORDS = {"airdrop", "camp", "crate", "event", "airport", "garrison", "compound", "outpost"}
		ESP.Register("MiniTargets", function()
			local out = {}
			for p in pairs(prompts) do
				if p.Parent and isTerritory(p) then
					local part = promptPart(p)
					if part and not (base.center and inMyBase(part)) then
						local txt = promptText(p)
						local kind, col = kindOf(txt)
						local par = p.Parent
						local nm = (p.ObjectText ~= "" and p.ObjectText) or (par and par.Name) or kind
						out[#out + 1] = {part = part, hl = (par and par:IsA("Model")) and par or nil,
							text = "[" .. kind .. "] " .. nm, color = col}
					end
				end
			end
			local now = os.clock()
			if now - nameCache.t > 4 then
				nameCache.t = now
				local list = {}
				for _, c in ipairs(workspace:GetChildren()) do
					if not c:IsA("Terrain") and not c:IsA("Camera") and not isPlayerChar(c) then
						local function check(i)
							if #list >= 30 then return end
							if (i:IsA("Model") or i:IsA("BasePart")) and hasAny(string.lower(i.Name), NAME_WORDS) then
								local part = i:IsA("BasePart") and i or unitPart(i)
								if part and not (base.center and inMyBase(part)) then list[#list + 1] = {inst = i, part = part} end
							end
						end
						check(c)
						for _, g in ipairs(c:GetChildren()) do check(g) end
					end
				end
				nameCache.list = list
			end
			for _, e in ipairs(nameCache.list) do
				if e.part.Parent then
					local kind, col = kindOf(string.lower(e.inst.Name))
					out[#out + 1] = {part = e.part, hl = e.inst:IsA("Model") and e.inst or nil,
						text = "[" .. kind .. "] " .. e.inst.Name, color = col}
				end
			end
			return out
		end)

		-- ======================================================
		-- G) INTERFACE: Quick | Merge | Economy | Combat | ESP | Tools
		-- ======================================================
		local function simple(sec, key, title, desc)
			ctl[key] = AddToggle(sec, title, desc, false, function(on)
				cfg[key] = on
				if on then ensureButtonTracking() end
			end)
		end

		-- ---------- QUICK (painel principal: liga e anda normalmente) ----------
		local pq = ctx.Page("Quick", ctx.Icons.Bolt)
		do
			local qs = pq.Section("Main Automation")
			ctl.Merge = AddToggle(qs, "Auto Merge", "Walks (no teleport) to equal units in your base", false, function(on)
				cfg.Merge = on
				if on then
					ensureBase()
					if cfg.AutoLearn and not learned then startLearning(600, true) end
				end
			end)
			-- quiet = true: Auto Upgrade não mostra pop-up nem toca som
			ctl.Upgrade = AddToggle(qs, "Auto Upgrade", "Silent. No menu opens on your screen", false, function(on)
				cfg.Upgrade = on
				if on then ensureButtonTracking() end
			end, true)
			-- quiet = true: Auto Rebirth não mostra pop-up nem toca som
			ctl.Rebirth = AddToggle(qs, "Auto Rebirth", "Silent. Careful: resets your progress", false, function(on)
				cfg.Rebirth = on
				if on then ensureButtonTracking() end
			end, true)
			simple(qs, "Collect", "Auto Collect / Claim", "Silent claim of rewards")
			simple(qs, "Buy", "Auto Buy / Spawn Units", "Silent buy and spawn")

			local qi = pq.Section("Live Status")
			Refs.qBase   = AddInfo(qi, "My base", "Not set")
			Refs.qPairs  = AddInfo(qi, "Pairs ready", "0")
			Refs.qMerges = AddInfo(qi, "Merges done", "0")
			Refs.qClicks = AddInfo(qi, "Silent clicks", "0")
			AddText(qi, "Ligue Auto Merge e ande normalmente: o personagem anda sozinho até os pares da sua base (sem teleporte). Se você mexer no movimento, ele pausa e retoma quando você parar. Rebirth e Upgrade clicam por trás, sem nada aparecer na tela.")
		end

		-- ---------- MERGE ----------
		local pm = ctx.Page("Merge", ctx.Icons.Merge)
		do
			local afk = pm.Section("AFK Farm")
			local AFK_KEYS = {"Merge", "Upgrade", "Collect", "Buy", "Equip", "Spin"}
			ctl.AFK = AddToggle(afk, "AFK Farm Mode", "Anti-AFK + merge, buy, upgrade, collect, equip, spin", false, function(on)
				ctx.SetAntiAfk(on)
				for _, key in ipairs(AFK_KEYS) do
					cfg[key] = on
					ctl[key].Set(on, true)
				end
				cfg.Rebirth = on and cfg.AfkRebirth
				ctl.Rebirth.Set(cfg.Rebirth, true)
				if on then
					ensureButtonTracking()
					ensureBase()
					if cfg.AutoLearn and not learned then startLearning(600, true) end
				end
				if on and not (firesignal or getconnections or VIM) then
					ctx.Notify("AFK Mode", "Executor can't click UI buttons", 4)
				end
			end)
			AddToggle(afk, "AFK includes Rebirth", "Also auto rebirths while AFK is on", false, function(on)
				cfg.AfkRebirth = on
				if ctl.AFK.Get() then cfg.Rebirth = on; ctl.Rebirth.Set(on, true) end
			end)
			AddToggle(afk, "AFK FPS Boost", "Lowest graphics while idle", false, function(on) ctx.SetLowGfx(on) end)
			AddToggle(afk, "AFK Disable 3D Rendering", "Black screen, minimum CPU/GPU", false, function(on) ctx.SetNoRender(on) end)

			local am = pm.Section("Auto Merge")
			AddSlider(am, "Merge delay", 0.03, 1.5, cfg.MergeDelay, 0.01, "s", function(v) cfg.MergeDelay = v end)
			AddSlider(am, "Walk timeout", 4, 20, cfg.WalkTimeout, 1, "s", function(v) cfg.WalkTimeout = v end)
			AddButton(am, "Merge Now", function()
				if ensureBase() then mergeOnce = true else ctx.Notify("Merge", "Set your base first", 3) end
			end)
			Refs.mUnits  = AddInfo(am, "Units in my base", "0")
			Refs.mPairs  = AddInfo(am, "Pairs found", "0")
			Refs.mMerges = AddInfo(am, "Merges done", "0")
			Refs.mFails  = AddInfo(am, "Failed attempts", "0")

			local mb = pm.Section("My Base (merge only here)")
			Refs.mBase = AddInfo(mb, "Base", "Not set")
			AddButton(mb, "Detect Base (auto)", function()
				base.center, base.model = nil, nil
				if detectBase() then ctx.Notify("Base", "Found: " .. base.label, 3)
				else ctx.Notify("Base", "Not found. Use Set Base Here", 4) end
			end)
			AddButton(mb, "Set Base Here", function()
				if setBaseHere() then ctx.Notify("Base", "Base set at your position", 3)
				else ctx.Notify("Base", "Character not found", 2) end
			end)
			AddSlider(mb, "Base radius (manual)", 20, 250, cfg.BaseRadius, 5, " st", function(v)
				cfg.BaseRadius = v
				if base.center and not base.model then base.radius = v end
			end)
			AddText(mb, "Fique no centro da sua base e toque em Set Base Here se a detecção automática não achar. Só unidades dentro dessa área são mescladas. Depois disso você pode andar para onde quiser.")

			local mm = pm.Section("Method")
			AddToggle(mm, "Use learned remote", "Fastest: replays the real merge call", true, function(on) cfg.UseRemote = on end)
			AddToggle(mm, "Auto-learn remote", "Learns it when you merge by hand once", true, function(on) cfg.AutoLearn = on end)
			Refs.mRemote = AddInfo(mm, "Learned remote", learned and learned.remote and learned.remote.Name or "None")
			AddButton(mm, "Learn Merge Remote", function() startLearning(20, false) end)
			AddButton(mm, "Forget learned remote", function()
				learned = nil
				env.__ShadowMiniLearn = nil
				Refs.mRemote.Text = "None"
			end)
			AddToggle(mm, "Drag unit onto target", "Experimental: moves unit A onto unit B", false, function(on) cfg.Drag = on end)
			AddInput(mm, "Unit names (optional)", "ex: soldier, tank", "", function(text)
				cfg.UnitWords = splitWords(text)
				scan.t = 0
				ctx.Notify("Units", #cfg.UnitWords > 0 and (#cfg.UnitWords .. " filter(s) set") or "Auto-detect units", 2)
			end)
			ctl.MergeBtn = AddToggle(mm, "Also click UI Merge button", "If the game has a Merge button", false, function(on)
				cfg.MergeBtn = on
				if on then ensureButtonTracking() end
			end)
		end

		-- ---------- ECONOMY ----------
		local pe = ctx.Page("Economy", ctx.Icons.Economy)
		do
			local up = pe.Section("Auto Buy Upgrades")
			AddInput(up, "Only these upgrades", "ex: spawn level, max", "", function(text)
				cfg.UpgradeWords = splitWords(text)
				ctx.Notify("Upgrades", #cfg.UpgradeWords > 0 and (#cfg.UpgradeWords .. " filter(s)") or "All upgrades", 2)
			end)
			AddSlider(up, "Upgrade delay", 0.2, 5, cfg.UpgradeEvery, 0.1, "s", function(v) cfg.UpgradeEvery = v end)
			AddButton(up, "Upgrade Now", function() clickNow("Upgrade") end)

			local rb = pe.Section("Auto Rebirth")
			AddSlider(rb, "Rebirth delay", 1, 300, cfg.RebirthEvery, 1, "s", function(v) cfg.RebirthEvery = v end)
			AddButton(rb, "Rebirth Now", function() clickNow("Rebirth") end)
			AddText(rb, "Ligue Auto Upgrade e Auto Rebirth na aba Quick. Eles rodam em silêncio. Depois do clique em Rebirth o script também confirma o diálogo (Confirm / Yes).")

			local sl = pe.Section("Silent Mode")
			AddToggle(sl, "Silent clicks", "Works with the game menu closed", true, function(on) cfg.Silent = on end)
			AddToggle(sl, "Mouse fallback", "Virtual mouse if no signal works (visible buttons only)", false, function(on) cfg.MouseFallback = on end)
			AddText(sl, "No modo silencioso o script aciona os botões por trás da interface: nada abre na tela e seu mouse/dedo continua livre para andar. Deixe o Mouse fallback desligado, ele usa clique de mouse real.")

			local ex = pe.Section("Extras")
			simple(ex, "Equip", "Auto Equip Best", "Clicks equip best / equip all")
			simple(ex, "Spin", "Auto Spin", "Free spins only (Robux ones are skipped)")
			AddSlider(ex, "Action delay", 0.2, 3, cfg.ActionDelay, 0.1, "s", function(v) cfg.ActionDelay = v end)
			AddText(ex, "Botões ligados a Robux, gamepass, VIP ou presentes são sempre ignorados.")
		end

		-- ---------- COMBAT ----------
		local pc = ctx.Page("Combat", ctx.Icons.Combat)
		do
			local dp = pc.Section("Auto Deploy")
			ctl.Deploy = AddToggle(dp, "Auto Deploy Army", "Clicks the deploy button", false, function(on)
				cfg.Deploy = on
				if on then ensureButtonTracking() end
			end)
			AddSlider(dp, "Deploy delay", 1, 30, cfg.DeployEvery, 0.5, "s", function(v) cfg.DeployEvery = v end)
			AddButton(dp, "Deploy Now", function() clickNow("Deploy") end)
			AddButton(dp, "Reset Army", function() clickNow("Reset") end)

			local at = pc.Section("Auto Base Attack")
			Refs.aStatus = AddInfo(at, "Status", "Idle")
			Refs.aTarget = AddInfo(at, "Target", "None")
			Refs.aCaps   = AddInfo(at, "Captures", "0")
			ctl.Attack = AddToggle(at, "Auto Base Attack", "Goes to targets and captures them", false, function(on)
				cfg.Attack = on
				if on then
					ensurePromptTracking()
					if not fireproximityprompt then ctx.Notify("Attack", "Executor lacks fireproximityprompt", 4) end
				end
			end)
			AddToggle(at, "Target: Event bases (Airdrop)", "Highest priority", true, function(on) cfg.AtkEvent = on end)
			AddToggle(at, "Target: Camps", "Camp / outpost prompts", true, function(on) cfg.AtkCamp = on end)
			AddToggle(at, "Target: Other territories", "Any other capture prompt", false, function(on) cfg.AtkBase = on end)
			AddToggle(at, "Teleport to target", "Off = only captures if you are near", true, function(on) cfg.AtkTeleport = on end)
			AddToggle(at, "Keep army deployed", "Clicks Deploy while attacking", true, function(on) cfg.KeepDeployed = on end)
			AddSlider(at, "Hold until captured (max)", 3, 40, cfg.AtkTimeout, 1, "s", function(v) cfg.AtkTimeout = v end)

			local cp = pc.Section("Auto Capture Nearby")
			ctl.AutoCapture = AddToggle(cp, "Auto Capture", "Triggers prompts around you", false, function(on)
				if on and not fireproximityprompt then
					ctx.Notify("Auto Capture", "Executor lacks fireproximityprompt", 4)
					ctl.AutoCapture.Set(false, true)
					return
				end
				cfg.AutoCapture = on
				if on then ensurePromptTracking() end
			end)
			AddSlider(cp, "Capture range", 5, 60, cfg.Range, 1, " st", function(v) cfg.Range = v end)
			AddToggle(cp, "Instant Interact", "Removes hold time on prompts", false, function(on)
				cfg.Instant = on
				if on then
					ensurePromptTracking()
					for p in pairs(prompts) do applyInstant(p) end
				else
					restoreInstant()
				end
			end)
		end

		-- ---------- ESP ----------
		local px = ctx.Page("ESP", ctx.Icons.Search)
		do
			local es = px.Section("ESP")
			Refs.espPlayersMini = AddToggle(es, "Players ESP", "Highlight + name + distance", false, function(on)
				ESP.SetEnabled("Players", on)
				if Refs.espPlayers then Refs.espPlayers.Set(on, true) end
			end)
			AddToggle(es, "Merge Pairs ESP", "Glows equal units in your base (same color = pair)", false, function(on)
				if on then ensureBase() end
				ESP.SetEnabled("MiniPairs", on)
			end)
			AddToggle(es, "Targets / Events ESP", "Airdrops, camps and territories", false, function(on)
				if on then ensurePromptTracking() end
				ESP.SetEnabled("MiniTargets", on)
			end)
			AddSlider(es, "ESP max distance", 100, 2000, ESP.MaxDist, 50, "", function(v) ESP.MaxDist = v end)
			AddText(es, "Cores: dourado = evento/airdrop, vermelho = camp, rosa = território. Pares de merge brilham na mesma cor.")
		end

		-- ---------- TOOLS ----------
		local pt = ctx.Page("Tools", ctx.Icons.Shield)
		do
			local cs = pt.Section("Custom Click")
			ctl.Custom = AddToggle(cs, "Custom Auto Click", "Clicks any button with your keywords", false, function(on)
				cfg.Custom = on
				if on then ensureButtonTracking() end
			end)
			AddInput(cs, "Keywords", "ex: claim, daily", "", function(text)
				cfg.CustomWords = splitWords(text)
				ctx.Notify("Custom Click", #cfg.CustomWords .. " keyword(s) set", 2)
			end)

			local st = pt.Section("Status")
			Refs.sClicks = AddInfo(st, "Buttons clicked", "0")
			Refs.sRebirth = AddInfo(st, "Rebirth clicks", "0")
			Refs.sButtons = AddInfo(st, "Buttons tracked", "0")
			Refs.sPrompts = AddInfo(st, "Prompts tracked", "0")

			local dev = pt.Section("Developer Tools")
			local function dump(out, name)
				table.sort(out)
				local text = table.concat(out, "\n")
				if ctx.copy then
					ctx.copy(text)
					ctx.Notify(name, #out .. " entries copied", 3)
				else
					print(text)
					ctx.Notify(name, #out .. " entries printed in the console", 3)
				end
			end
			AddButton(dev, "Scan Buttons", function()
				ensureButtonTracking()
				local out, now = {}, os.clock()
				for b, info in pairs(buttons) do
					if b.Parent then
						refreshInfo(b, info, now)
						table.insert(out, (isShown(b) and "[visible] " or "[hidden] ") .. b:GetFullName() .. " | " .. info.text)
					end
				end
				dump(out, "Scan Buttons")
			end)
			AddButton(dev, "Scan My Base Units", function()
				if not ensureBase() then return end
				local sc = scanUnits(true)
				local out = {}
				for key, list in pairs(sc.groups) do
					table.insert(out, #list .. "x | " .. key .. " | " .. list[1]:GetFullName())
				end
				dump(out, "Base Units")
			end)
			AddButton(dev, "Scan Remotes", function()
				local out = {}
				for _, d in ipairs(game:GetService("ReplicatedStorage"):GetDescendants()) do
					if d:IsA("RemoteEvent") or d:IsA("RemoteFunction") then
						table.insert(out, d.ClassName .. ": " .. d:GetFullName())
					end
				end
				dump(out, "Scan Remotes")
			end)
			AddButton(dev, "Reset merge cooldowns", function()
				cooldown = setmetatable({}, {__mode = "k"})
				failCount = setmetatable({}, {__mode = "k"})
				scan.t = 0
				ctx.Notify("Merge", "Cooldowns cleared", 2)
			end)
			AddText(dev, "Se o merge não achar pares: rode Scan My Base Units e me mande o resultado, ou preencha Unit names na aba Merge.")
		end

		-- Atualiza os contadores 1x/s, só com a aba visível
		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(1)
				if ctx.IsVisible() then
					local baseText = base.center and base.label or "Not set"
					Refs.mUnits.Text = tostring(scan.units)
					Refs.mPairs.Text = tostring(scan.pairs)
					Refs.mMerges.Text = tostring(stats.merges)
					Refs.mFails.Text = tostring(stats.fails)
					Refs.mBase.Text = baseText
					Refs.qBase.Text = baseText
					Refs.qPairs.Text = tostring(scan.pairs)
					Refs.qMerges.Text = tostring(stats.merges)
					Refs.qClicks.Text = tostring(stats.clicks)
					Refs.aStatus.Text = atk.status
					Refs.aTarget.Text = atk.target
					Refs.aCaps.Text = tostring(stats.captures)
					Refs.sClicks.Text = tostring(stats.clicks)
					Refs.sRebirth.Text = tostring(stats.rebirths)
					local bc, pc2 = 0, 0
					for _ in pairs(buttons) do bc = bc + 1 end
					for _ in pairs(prompts) do pc2 = pc2 + 1 end
					Refs.sButtons.Text = tostring(bc)
					Refs.sPrompts.Text = tostring(pc2)
				end
			end
		end)

		ctx.OnUnload(function()
			running = false
			recording = false
			mergeOnce = false
			env.__ShadowHookFn = nil
			for _, k in ipairs(DUE_KEYS) do cfg[k] = false end
			cfg.Merge, cfg.Attack, cfg.AutoCapture = false, false, false
			stopWalk()
			restoreInstant()
		end)
	end,
})

-- Detecção do jogo atual (só carrega o script se ESTE for o jogo dele).
-- Em qualquer outro jogo nenhuma aba de jogo é criada.
-- Camadas: 1) IDs exatos  2) nome do jogo (com tentativas)  3) conteúdo do jogo
task.spawn(function()
	-- 1) PlaceId / GameId
	for _, def in ipairs(GameModules) do
		if matchesGame(def, nil) then loadGame(def) end
	end

	-- 2) Nome vindo do Roblox (tenta algumas vezes: às vezes a primeira chamada falha)
	local gameName
	for _ = 1, 4 do
		local ok, info = pcall(function() return game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId) end)
		if ok and info and info.Name then gameName = info.Name break end
		task.wait(1.5)
	end
	if not ScreenGui.Parent then return end
	Refs.gameName.Text = gameName or game.Name or "Unknown"
	for _, def in ipairs(GameModules) do
		if matchesGame(def, gameName) then loadGame(def) end
	end

	-- 3) Plano B: o jogo carrega aos poucos, então tenta de novo por ~1 minuto
	for _ = 1, 12 do
		if not ScreenGui.Parent then return end
		local pending = false
		for _, def in ipairs(GameModules) do
			if def.Detect and not loadedGames[def.Name] then
				pending = true
				local ok, found = pcall(def.Detect)
				if ok and found then loadGame(def) end
			end
		end
		if not pending then break end
		task.wait(5)
	end

	if next(loadedGames) == nil and ScreenGui.Parent then
		Refs.gameScript.Text = "None for this game"
	end
end)

-- ================================================================= --
-- 13. LOOPS, ABRIR/FECHAR, ARRASTE, UNLOAD E INICIALIZAÇÃO
-- ================================================================= --
do
	-- Pulo infinito
	track(UserInputService.JumpRequest:Connect(function()
		if State.InfJump then
			local hum = getHum()
			if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
		end
	end))

	-- Noclip com lista em cache
	track(RunService.Stepped:Connect(function()
		if not State.Noclip then return end
		local now = os.clock()
		if now - Mv.refresh > 1.5 then
			Mv.refresh = now
			Mv.rebuildNoclip()
		end
		local parts = Mv.parts
		for i = 1, #parts do
			local p = parts[i]
			if p.CanCollide then
				Mv.touched[p] = true
				p.CanCollide = false
			end
		end
	end))

	-- Heartbeat: velocidade, pulo, fly, iluminação (só escreve se mudou)
	track(RunService.Heartbeat:Connect(function(dt)
		if State.SpeedOn or State.JumpOn then
			local hum = getHum()
			if hum then
				if State.SpeedOn then
					if Original.Speed == nil then Original.Speed = hum.WalkSpeed end
					if hum.WalkSpeed ~= State.Speed then hum.WalkSpeed = State.Speed end
				end
				if State.JumpOn then
					if Original.Jump == nil then
						Original.JumpIsPower = hum.UseJumpPower
						Original.Jump = hum.UseJumpPower and hum.JumpPower or hum.JumpHeight
					end
					if hum.UseJumpPower then
						if hum.JumpPower ~= State.Jump then hum.JumpPower = State.Jump end
					else
						local target = (State.Jump * State.Jump) / (2 * workspace.Gravity)
						if hum.JumpHeight ~= target then hum.JumpHeight = target end
					end
				end
			end
		end

		if State.Fly then
			local fl = Mv.fly
			if fl and fl.bv and fl.bv.Parent then
				local cam = workspace.CurrentCamera
				local root = getRoot()
				if cam and root then
					local mv = Mv.moveVector()
					local dir = cam.CFrame.RightVector * mv.X - cam.CFrame.LookVector * mv.Z
					if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.yAxis end
					if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) or UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
						dir = dir - Vector3.yAxis
					end
					if dir.Magnitude > 1 then dir = dir.Unit end
					Mv.vel = Mv.vel:Lerp(dir * State.FlySpeed, 1 - math.exp(-12 * dt))
					fl.bv.Velocity = Mv.vel
					local flat = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
					if flat.Magnitude > 0.01 then fl.bg.CFrame = CFrame.lookAt(root.Position, root.Position + flat) end
				end
			elseif not fl then
				Mv.startFly()
			end
		end

		if State.Fullbright and Vis.fb then
			if Lighting.Brightness ~= 2 then Lighting.Brightness = 2 end
			if Lighting.ClockTime ~= 14 then Lighting.ClockTime = 14 end
			if Lighting.GlobalShadows then Lighting.GlobalShadows = false end
			local amb = Color3.fromRGB(178, 178, 178)
			if Lighting.Ambient ~= amb then Lighting.Ambient = amb end
			if Lighting.OutdoorAmbient ~= amb then Lighting.OutdoorAmbient = amb end
		end
		if State.NoFog and Vis.fog then
			if Lighting.FogStart ~= 1e6 then Lighting.FogStart = 1e6 end
			if Lighting.FogEnd ~= 1e6 then Lighting.FogEnd = 1e6 end
			for atm in pairs(Vis.fog.Atmos) do
				if atm.Parent and atm.Density ~= 0 then atm.Density = 0 end
			end
		end
	end))

	-- RenderStepped: FOV, neve, FPS, Performance Guard e estatísticas
	local frames, lastStat, snowAcc = 0, os.clock(), 0
	track(RunService.RenderStepped:Connect(function(dt)
		if State.FovOn then
			local cam = workspace.CurrentCamera
			if cam and cam.FieldOfView ~= State.Fov then cam.FieldOfView = State.Fov end
		end

		if isOpen and State.Snow and not Perf.Lite then
			snowAcc = snowAcc + dt
			if snowAcc >= 1 / 24 then
				updateSnow(snowAcc)
				snowAcc = 0
			end
		else
			snowAcc = 0
		end

		frames = frames + 1
		local now = os.clock()
		if now - lastStat >= 1 then
			guardTick(frames)
			if isOpen then
				Refs.fps.Text = tostring(frames)
				Refs.perfFps.Text = tostring(frames)
				Refs.perfState.Text = Perf.Status
				local elapsed = os.time() - startTime
				if elapsed >= 3600 then
					Refs.session.Text = string.format("%dh %02dm", math.floor(elapsed / 3600), math.floor((elapsed % 3600) / 60))
				else
					Refs.session.Text = string.format("%02dm %02ds", math.floor(elapsed / 60), elapsed % 60)
				end
				local count = #Players:GetPlayers() .. "/" .. Players.MaxPlayers
				Refs.players.Text = count
				Refs.srvPlayers.Text = count
				local age = math.floor(workspace.DistributedGameTime)
				local ageText = string.format("%dh %02dm %02ds", math.floor(age / 3600), math.floor((age % 3600) / 60), age % 60)
				Refs.age.Text = ageText
				Refs.srvAge.Text = ageText
				local ok, ping = pcall(function() return game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue() end)
				if not ok or not ping then ping = LocalPlayer:GetNetworkPing() * 1000 end
				Refs.ping.Text = math.floor(ping) .. "ms"
			end
			frames = 0
			lastStat = now
		end
	end))

	-- Loop do ESP (4x/s, mais lento no modo Lite)
	task.spawn(function()
		while ScreenGui.Parent do
			local any = false
			for _, on in pairs(ESP.Enabled) do
				if on then any = true break end
			end
			if any then pcall(ESP.Tick) end
			task.wait(Perf.Lite and 0.6 or 0.25)
		end
	end)

	track(LocalPlayer.CharacterAdded:Connect(function(char)
		cHum, cRoot = nil, nil
		Original.Speed, Original.Jump = nil, nil
		Mv.touched = {}
		table.clear(Mv.parts)
		Mv.refresh = 0
		Mv.fly = nil
		Mv.vel = Vector3.zero
		if State.Fly then
			task.spawn(function()
				char:WaitForChild("Humanoid", 8)
				char:WaitForChild("HumanoidRootPart", 8)
				task.wait(0.4)
				if State.Fly and LocalPlayer.Character == char then Mv.startFly() end
			end)
		end
	end))

	-- Abrir / fechar
	local animToken = 0
	local function setOpen(state)
		if state == isOpen then return end
		isOpen = state
		animToken = animToken + 1
		local token = animToken
		Sfx.Play(isOpen and "Open" or "Close")
		if isOpen then
			currentScale = computeScale()
			hubCenter = clampCenter(hubCenter)
			MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
			MainFrame.Visible = true
			SnowContainer.Visible = State.Snow and not Perf.Lite
			HubScale.Scale = currentScale * 0.9
			MainFrame.GroupTransparency = 1
			tween(MainFrame, 0.35, {GroupTransparency = 0})
			tween(HubScale, 0.4, {Scale = currentScale}, Enum.EasingStyle.Back)
			tween(FloatingBtn, 0.4, {Rotation = 180, BackgroundColor3 = COLORS.Card})
			tween(BtnStroke, 0.4, {Color = COLORS.AccentGlow, Transparency = 0})
		else
			tween(MainFrame, 0.25, {GroupTransparency = 1})
			tween(HubScale, 0.25, {Scale = currentScale * 0.9}, Enum.EasingStyle.Quart, Enum.EasingDirection.In)
			tween(FloatingBtn, 0.4, {Rotation = 0, BackgroundColor3 = COLORS.Sidebar})
			tween(BtnStroke, 0.4, {Color = COLORS.Accent, Transparency = 0.3})
			task.delay(0.27, function()
				if animToken == token and not isOpen then
					MainFrame.Visible = false
					SnowContainer.Visible = false
				end
			end)
		end
	end

	MinimizeBtn.MouseButton1Click:Connect(function() setOpen(false) end)

	track(ScreenGui:GetPropertyChangedSignal("AbsoluteSize"):Connect(function()
		currentScale = computeScale()
		if isOpen then HubScale.Scale = currentScale end
		hubCenter = clampCenter(hubCenter)
		MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
	end))

	-- Tecla do menu (PC)
	track(UserInputService.InputBegan:Connect(function(input, gp)
		if listeningKey and input.UserInputType == Enum.UserInputType.Keyboard then
			listeningKey = false
			if input.KeyCode ~= Enum.KeyCode.Escape then toggleKey = input.KeyCode end
			if Refs.keyBtn then
				Refs.keyBtn.Text = toggleKey.Name
				tween(Refs.keyBtn, 0.15, {BackgroundColor3 = COLORS.CardHeader})
			end
			Notify("Keybind", "Menu key: " .. toggleKey.Name, 2)
			return
		end
		if not gp and input.KeyCode == toggleKey then setOpen(not isOpen) end
	end))

	-- Arraste (mouse e toque)
	local function makeDrag(handle, onStart, onMove, onEnd)
		local active, dragInput, startPos = false, nil, Vector2.zero
		handle.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				active, dragInput = true, input
				startPos = Vector2.new(input.Position.X, input.Position.Y)
				onStart()
			end
		end)
		track(UserInputService.InputChanged:Connect(function(input)
			if active and (input.UserInputType == Enum.UserInputType.MouseMovement or input == dragInput) then
				onMove(Vector2.new(input.Position.X, input.Position.Y) - startPos)
			end
		end))
		track(UserInputService.InputEnded:Connect(function(input)
			if active and (input == dragInput or input.UserInputType == Enum.UserInputType.MouseButton1) then
				active = false
				onEnd()
			end
		end))
	end

	local hubMoved, hubStart, ghost = false, hubCenter, hubCenter
	local function attachHubDrag(handle)
		makeDrag(handle,
			function() hubMoved = false; hubStart = hubCenter; ghost = hubCenter end,
			function(delta)
				if not hubMoved and delta.Magnitude > 3 then
					hubMoved = true
					InputBlocker.Visible = true
					DragGhost.Size = UDim2.fromOffset(hubPixelSize().X, hubPixelSize().Y)
					DragGhost.Visible = true
				end
				if hubMoved then
					ghost = clampCenter(hubStart + delta)
					DragGhost.Position = UDim2.fromOffset(ghost.X, ghost.Y)
				end
			end,
			function()
				if hubMoved then
					hubMoved = false
					hubCenter = ghost
					DragGhost.Visible = false
					InputBlocker.Visible = false
					tween(MainFrame, 0.3, {Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)})
				end
			end
		)
	end
	attachHubDrag(Sidebar)
	attachHubDrag(TopHeader)

	local fbMoved, fbStart = false, Vector2.zero
	makeDrag(FloatingBtn,
		function()
			fbMoved = false
			fbStart = Vector2.new(FloatingBtn.AbsolutePosition.X, FloatingBtn.AbsolutePosition.Y)
		end,
		function(delta)
			if not fbMoved and delta.Magnitude > 6 then
				fbMoved = true
				InputBlocker.Visible = true
			end
			if fbMoved then
				local s = screenSize()
				local p = fbStart + delta
				p = Vector2.new(math.clamp(p.X, 0, math.max(0, s.X - 48)), math.clamp(p.Y, 0, math.max(0, s.Y - 48)))
				FloatingBtn.Position = UDim2.fromOffset(p.X, p.Y)
			end
		end,
		function()
			InputBlocker.Visible = false
			if not fbMoved then setOpen(not isOpen) end
			fbMoved = false
		end
	)

	-- Unload completo
	UnloadHub = function()
		for _, fn in ipairs(cleanupFns) do pcall(fn) end
		env.__ShadowHookFn = nil
		State.Fly = false
		Mv.stopFly()
		Mv.restoreSpeed()
		Mv.restoreJump()
		Mv.restoreNoclip()
		Vis.restoreFullbright()
		Vis.restoreFog()
		setAntiLag(false)
		setUltra(false)
		setLowGfx(false)
		setAntiAfk(false)
		setAutoRejoin(false)
		pcall(function() RunService:Set3dRenderingEnabled(true) end)
		if Original.Fov and workspace.CurrentCamera then workspace.CurrentCamera.FieldOfView = Original.Fov end
		pcall(function() LocalPlayer.CameraMaxZoomDistance = origMaxZoom end)
		for _, c in ipairs(connections) do pcall(function() c:Disconnect() end) end
		connections = {}
		env.__ShadowHubUnload = nil
		if ESPFolder then ESPFolder:Destroy() end
		if ScreenGui then ScreenGui:Destroy() end
	end
	env.__ShadowHubUnload = UnloadHub

	-- Inicialização
	currentScale = computeScale()
	local s = screenSize()
	hubCenter = clampCenter(Vector2.new(s.X / 2, s.Y / 2))
	MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
	HubScale.Scale = currentScale
	setActiveTab("Home")

	task.delay(0.3, function()
		setOpen(true)
		Notify("Dark Shadow Hub", IS_TOUCH and "Loaded. Tap the floating button to toggle." or ("Loaded. Press " .. toggleKey.Name .. " to toggle."), 3)
	end)
end
