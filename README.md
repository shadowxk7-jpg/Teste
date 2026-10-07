-- ================================================================= --
--   DARK SHADOW HUB v3.0  |  LUXURY EDITION                          --
--   + Aba dedicada: Merge a Mini Army (AFK Farm completo)            --
--   Otimizado: cache de personagem, rastreio sob demanda, menos GC   --
-- ================================================================= --
--  ÍNDICE
--   1. Serviços e ambiente
--   2. Utilitários e tema
--   3. Estado global
--   4. Interface base (ScreenGui, neve, janela, sidebar)
--   5. Framework de abas e sub-abas
--   6. Notificações
--   7. Componentes (toggle, slider, botão, input, info, texto)
--   8. Busca
--   9. Abas: Home / Player / Main / Settings
--  10. Sistema de scripts por jogo
--  11. Jogo: Merge a Mini Army
--  12. Loops principais
--  13. Abrir/fechar, arraste, unload, inicialização
-- ================================================================= --

if not game:IsLoaded() then game.Loaded:Wait() end

-- ==========================================
-- 1. SERVIÇOS E AMBIENTE
-- ==========================================
local TweenService       = game:GetService("TweenService")
local UserInputService   = game:GetService("UserInputService")
local RunService         = game:GetService("RunService")
local Players            = game:GetService("Players")
local MarketplaceService = game:GetService("MarketplaceService")
local TeleportService    = game:GetService("TeleportService")
local HttpService        = game:GetService("HttpService")
local Lighting           = game:GetService("Lighting")
local VirtualUser        = game:GetService("VirtualUser")
local GuiService         = game:GetService("GuiService")
local Stats              = game:GetService("Stats")

local LocalPlayer = Players.LocalPlayer
local ParentGui   = (gethui and gethui()) or LocalPlayer:WaitForChild("PlayerGui")
local env         = (getgenv and getgenv()) or _G

-- Descarrega instância anterior (evita conexões duplicadas)
if env.__ShadowHubUnload then pcall(env.__ShadowHubUnload) end
if ParentGui:FindFirstChild("ShadowTechHub") then ParentGui.ShadowTechHub:Destroy() end

-- ==========================================
-- 2. UTILITÁRIOS E TEMA
-- ==========================================
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
}

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

local State = {
	Speed = 50, SpeedOn = false,
	Jump = 100, JumpOn = false,
	InfJump = false, Noclip = false,
	Fly = false, FlySpeed = 60,
	Fullbright = false, NoFog = false,
	FovOn = false, Fov = 70,
	AntiAfk = false, LowGfx = false,
	NoRender = false, AutoRejoin = false,
	Snow = true,
}
local Original = {}
local noclipTouched = {}
local noclipParts = {}
local noclipRefresh = 0
local flyObjs = nil
local flyVelocity = Vector3.zero
local controlsModule = nil
local startTime = os.time()
local origMaxZoom = LocalPlayer.CameraMaxZoomDistance

pcall(function()
	controlsModule = require(LocalPlayer:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 3)):GetControls()
end)

-- ==========================================
-- 4. INTERFACE BASE
-- ==========================================
local ScreenGui = create("ScreenGui", {
	Name = "ShadowTechHub",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	DisplayOrder = 999,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	Parent = ParentGui,
})

local InputBlocker = create("TextButton", {
	Name = "InputBlocker", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
	Text = "", Visible = false, Modal = true, ZIndex = 1, Parent = ScreenGui,
})

-- Neve (pool único, 24 FPS, só com o menu aberto)
local SnowContainer = create("Frame", {
	Name = "SnowContainer", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
	ClipsDescendants = true, Visible = false, ZIndex = 1, Parent = ScreenGui,
})

local flakes = {}
for i = 1, 28 do
	local size = math.random(3, 7)
	local frame = create("Frame", {
		Size = UDim2.fromOffset(size, size),
		BackgroundColor3 = Color3.fromRGB(255, 230, 245),
		BackgroundTransparency = math.random(15, 50) / 100,
		BorderSizePixel = 0, Parent = SnowContainer,
	})
	corner(frame, 8)
	flakes[i] = {
		frame = frame, x = math.random(), y = math.random(),
		speed = math.random(80, 220) / 1000,
		drift = math.random(10, 40) / 10,
		seed = math.random(1, 100),
	}
end

local function updateSnow(dt)
	local now = os.clock()
	for i = 1, #flakes do
		local f = flakes[i]
		f.y = f.y + f.speed * dt
		f.x = (f.x + math.sin(now * f.drift + f.seed) * 0.015 * dt) % 1
		if f.y > 1.05 then
			f.y = -0.05
			f.x = math.random()
		end
		f.frame.Position = UDim2.fromScale(f.x, f.y)
	end
end

-- Botão flutuante
local FloatingBtn = create("Frame", {
	Name = "FloatingButton", Size = UDim2.fromOffset(48, 48),
	Position = UDim2.new(0, 24, 0.25, 0),
	BackgroundColor3 = COLORS.Sidebar, BorderSizePixel = 0, Active = true,
	ZIndex = 10, Parent = ScreenGui,
})
corner(FloatingBtn, 14)
local BtnStroke = stroke(FloatingBtn, COLORS.Accent, 1.5, 0.3)
create("ImageLabel", {
	Size = UDim2.fromOffset(24, 24), AnchorPoint = Vector2.new(0.5, 0.5),
	Position = UDim2.fromScale(0.5, 0.5), BackgroundTransparency = 1,
	Image = ICONS.Logo, ImageColor3 = COLORS.Accent, Parent = FloatingBtn,
})

-- Janela principal (CanvasGroup => fade suave)
local HUB_W, HUB_H = 680, 430
local hubCenter = Vector2.new(0, 0)

local MainFrame = create("CanvasGroup", {
	Name = "MainHub", Size = UDim2.fromOffset(HUB_W, HUB_H),
	AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = COLORS.Background,
	BorderSizePixel = 0, Visible = false, Active = true, ZIndex = 5,
	GroupTransparency = 1, Parent = ScreenGui,
})
corner(MainFrame, 16)
stroke(MainFrame, COLORS.Stroke, 1.2)
local HubScale = create("UIScale", {Scale = 1, Parent = MainFrame})

local DragGhost = create("Frame", {
	Name = "DragGhost", AnchorPoint = Vector2.new(0.5, 0.5), Visible = false,
	BackgroundTransparency = 0.85, BackgroundColor3 = COLORS.Accent,
	BorderSizePixel = 0, ZIndex = 20, Parent = ScreenGui,
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
local Sidebar = create("Frame", {
	Size = UDim2.new(0, 150, 1, 0), BackgroundColor3 = COLORS.Sidebar,
	BorderSizePixel = 0, Active = true, Parent = MainFrame,
})
create("Frame", {
	AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromScale(1, 0),
	Size = UDim2.new(0, 1, 1, 0), BackgroundColor3 = COLORS.Stroke,
	BorderSizePixel = 0, Parent = Sidebar,
})

local LogoHeader = create("Frame", {Size = UDim2.new(1, 0, 0, 60), BackgroundTransparency = 1, Parent = Sidebar})
create("ImageLabel", {
	Size = UDim2.fromOffset(28, 28), Position = UDim2.new(0, 16, 0.5, -14),
	BackgroundTransparency = 1, Image = ICONS.Logo, ImageColor3 = COLORS.Accent, Parent = LogoHeader,
})
label({Size = UDim2.new(1, -54, 0, 16), Position = UDim2.new(0, 52, 0.5, -15), Text = "DARK SHADOW",
	Font = Enum.Font.GothamBlack, TextSize = 12, Parent = LogoHeader})
label({Size = UDim2.new(1, -54, 0, 12), Position = UDim2.new(0, 52, 0.5, 2), Text = "HUB  v3.0",
	TextSize = 10, TextColor3 = COLORS.Accent, Parent = LogoHeader})

local TabsContainer = create("Frame", {
	Size = UDim2.new(1, -16, 0, 270), Position = UDim2.new(0, 8, 0, 66),
	BackgroundTransparency = 1, Parent = Sidebar,
})
create("UIListLayout", {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 6), Parent = TabsContainer})

-- Perfil
local ProfileCard = create("Frame", {
	Size = UDim2.new(1, -16, 0, 48), Position = UDim2.new(0, 8, 1, -56),
	BackgroundTransparency = 1, Parent = Sidebar,
})
local AvatarFrame = create("Frame", {
	Size = UDim2.fromOffset(36, 36), Position = UDim2.new(0, 2, 0.5, -18),
	BackgroundTransparency = 1, Parent = ProfileCard,
})
local AvatarImg = create("ImageLabel", {
	Size = UDim2.fromScale(1, 1), BackgroundColor3 = COLORS.Card, Parent = AvatarFrame,
})
corner(AvatarImg, 18)
local StatusDot = create("Frame", {
	Size = UDim2.fromOffset(10, 10), Position = UDim2.new(1, -8, 1, -8),
	BackgroundColor3 = COLORS.Green, BorderSizePixel = 0, ZIndex = 3, Parent = AvatarFrame,
})
corner(StatusDot, 5)
stroke(StatusDot, COLORS.Sidebar, 1.5)
label({Size = UDim2.new(1, -46, 0, 16), Position = UDim2.new(0, 44, 0, 8), Text = LocalPlayer.DisplayName,
	Font = Enum.Font.GothamBold, TextTruncate = Enum.TextTruncate.AtEnd, Parent = ProfileCard})
label({Size = UDim2.new(1, -46, 0, 14), Position = UDim2.new(0, 44, 0, 24), Text = "@" .. LocalPlayer.Name,
	TextSize = 10, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = ProfileCard})

-- Conteúdo e header
local ContentArea = create("Frame", {
	Size = UDim2.new(1, -150, 1, 0), Position = UDim2.new(0, 150, 0, 0),
	BackgroundTransparency = 1, Parent = MainFrame,
})

local TopHeader = create("Frame", {
	Size = UDim2.new(1, -20, 0, 52), Position = UDim2.new(0, 10, 0, 4),
	BackgroundTransparency = 1, Active = true, Parent = ContentArea,
})
local HeaderIcon = create("ImageLabel", {
	Size = UDim2.fromOffset(20, 20), Position = UDim2.new(0, 4, 0.5, -10),
	BackgroundTransparency = 1, ImageColor3 = COLORS.Accent, Image = ICONS.Home, Parent = TopHeader,
})
local HeaderTitle = label({
	Size = UDim2.new(0, 120, 1, 0), Position = UDim2.new(0, 32, 0, 0), Text = "Home",
	Font = Enum.Font.GothamBold, TextSize = 16, Parent = TopHeader,
})

local MinimizeBtn = create("TextButton", {
	Size = UDim2.fromOffset(28, 28), AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, 0, 0.5, 0), BackgroundColor3 = COLORS.Card,
	Text = "—", TextColor3 = COLORS.TextDark, TextSize = 13, Font = Enum.Font.GothamBold,
	AutoButtonColor = false, Parent = TopHeader,
})
corner(MinimizeBtn, 8)
MinimizeBtn.MouseEnter:Connect(function() tween(MinimizeBtn, 0.15, {BackgroundColor3 = COLORS.Danger, TextColor3 = COLORS.TextMain}) end)
MinimizeBtn.MouseLeave:Connect(function() tween(MinimizeBtn, 0.15, {BackgroundColor3 = COLORS.Card, TextColor3 = COLORS.TextDark}) end)

local SearchFrame = create("Frame", {
	Size = UDim2.fromOffset(170, 28), AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -36, 0.5, 0), BackgroundColor3 = COLORS.Card, Parent = TopHeader,
})
corner(SearchFrame, 8)
local SearchStroke = stroke(SearchFrame, COLORS.Stroke, 1)
create("ImageLabel", {
	Size = UDim2.fromOffset(14, 14), Position = UDim2.new(0, 9, 0.5, -7),
	BackgroundTransparency = 1, Image = ICONS.Search, ImageColor3 = COLORS.TextDark, Parent = SearchFrame,
})
local SearchBox = create("TextBox", {
	Size = UDim2.new(1, -32, 1, 0), Position = UDim2.new(0, 28, 0, 0), BackgroundTransparency = 1,
	Text = "", PlaceholderText = "Search...", PlaceholderColor3 = COLORS.TextDark,
	TextColor3 = COLORS.TextMain, TextSize = 11, Font = Enum.Font.GothamMedium,
	TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false, Parent = SearchFrame,
})
SearchBox.Focused:Connect(function() tween(SearchStroke, 0.2, {Color = COLORS.Accent}) end)
SearchBox.FocusLost:Connect(function() tween(SearchStroke, 0.2, {Color = COLORS.Stroke}) end)

local PagesContainer = create("Frame", {
	Size = UDim2.new(1, -20, 1, -64), Position = UDim2.new(0, 10, 0, 58),
	BackgroundTransparency = 1, Parent = ContentArea,
})

local NoResults = label({
	Size = UDim2.fromScale(1, 1), Text = "No results found", TextSize = 13,
	TextColor3 = COLORS.TextDark, TextXAlignment = Enum.TextXAlignment.Center,
	Visible = false, ZIndex = 0, Parent = PagesContainer,
})

-- ==========================================
-- 5. FRAMEWORK DE ABAS E SUB-ABAS
-- ==========================================
local tabs = {}     -- abas da sidebar
local groups = {}   -- grupos de sub-abas por aba

local function createScroll(parent)
	local s = create("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
		ScrollBarThickness = 3, ScrollBarImageColor3 = COLORS.Accent,
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
		Size = UDim2.fromOffset(18, 18), Position = UDim2.new(0, 14, 0.5, -9),
		BackgroundTransparency = 1, Image = iconId, ImageColor3 = COLORS.TextDark, Parent = btn,
	})
	local lbl = label({Size = UDim2.new(1, -44, 1, 0), Position = UDim2.new(0, 40, 0, 0), Text = name,
		TextColor3 = COLORS.TextDark, Parent = btn})

	local page = create("Frame", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, Parent = PagesContainer})
	local content = page
	if useScroll then content = createScroll(page) end

	tabs[name] = {Btn = btn, Page = page, Icon = icon, Line = line, Label = lbl, IconId = iconId, Active = false}

	btn.MouseButton1Click:Connect(function() setActiveTab(name) end)
	btn.MouseEnter:Connect(function() if not tabs[name].Active then tween(btn, 0.15, {BackgroundTransparency = 0.8}) end end)
	btn.MouseLeave:Connect(function() if not tabs[name].Active then tween(btn, 0.15, {BackgroundTransparency = 1}) end end)
	return content
end

-- Grupo de sub-abas dentro de uma aba (Main, Mini Army, ...)
local function createSubGroup(tabName, page)
	local bar = create("Frame", {Size = UDim2.new(1, 0, 0, 34), BackgroundTransparency = 1, Parent = page})
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
			Size = UDim2.fromOffset(104, 34), BackgroundColor3 = COLORS.SubTabBg, Text = "",
			AutoButtonColor = false, LayoutOrder = nextOrder(bar), Parent = bar,
		})
		corner(btn, 8)
		local icon = create("ImageLabel", {
			Size = UDim2.fromOffset(14, 14), Position = UDim2.new(0, 10, 0.5, -7),
			BackgroundTransparency = 1, Image = iconId, ImageColor3 = COLORS.TextDark, Parent = btn,
		})
		local lbl = label({Size = UDim2.new(1, -30, 1, 0), Position = UDim2.new(0, 30, 0, 0), Text = name,
			TextSize = 11, TextColor3 = COLORS.TextDark, Parent = btn})
		local holderPage = create("Frame", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, Parent = holder})
		local scroll = createScroll(holderPage)
		group.Subs[name] = {Btn = btn, Page = holderPage, Icon = icon, Label = lbl}
		btn.MouseButton1Click:Connect(function() group.Set(name) end)
		if not group.Active then group.Set(name) end
		return scroll
	end

	groups[tabName] = group
	return group
end

-- Ordem das abas na sidebar
local ORDER = {Home = 10, Player = 20, Main = 30, Game = 40, Settings = 100}

local HomeScroll     = createTab("Home", ICONS.Home, true, ORDER.Home)
local PlayerScroll   = createTab("Player", ICONS.Player, true, ORDER.Player)
local MainPage       = createTab("Main", ICONS.Main, false, ORDER.Main)
local SettingsScroll = createTab("Settings", ICONS.Settings, true, ORDER.Settings)

local MainGroup     = createSubGroup("Main", MainPage)
local VisualsScroll = MainGroup.Add("Visuals", ICONS.Merge)
local UtilityScroll = MainGroup.Add("Utility", ICONS.Shield)
local ServerScroll  = MainGroup.Add("Server", ICONS.Server)

-- ==========================================
-- 6. NOTIFICAÇÕES
-- ==========================================
local NotifHolder = create("Frame", {
	Name = "Notifications", Size = UDim2.new(0, 250, 1, -20), Position = UDim2.new(1, -262, 0, 10),
	BackgroundTransparency = 1, ZIndex = 50, Parent = ScreenGui,
})
create("UIListLayout", {
	VerticalAlignment = Enum.VerticalAlignment.Bottom, SortOrder = Enum.SortOrder.LayoutOrder,
	Padding = UDim.new(0, 8), Parent = NotifHolder,
})

local activeNotifs = 0
local function Notify(title, text, duration)
	if not notificationsEnabled then return end
	if activeNotifs >= 4 then return end -- evita acúmulo (lag)
	activeNotifs = activeNotifs + 1
	duration = duration or 2.5
	local wrapper = create("Frame", {
		Size = UDim2.new(1, 0, 0, 50), BackgroundTransparency = 1, ZIndex = 50,
		LayoutOrder = nextOrder(NotifHolder), Parent = NotifHolder,
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
		Font = Enum.Font.GothamBold, TextSize = 12, Parent = card})
	label({Size = UDim2.new(1, -28, 0, 14), Position = UDim2.new(0, 20, 0, 26), Text = text,
		TextSize = 10, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = card})

	tween(card, 0.35, {Position = UDim2.new(0, 0, 0, 0)}, Enum.EasingStyle.Back)
	task.delay(duration, function()
		if card.Parent then
			tween(card, 0.3, {GroupTransparency = 1, Position = UDim2.new(0.3, 0, 0, 0)})
			task.wait(0.32)
		end
		activeNotifs = math.max(0, activeNotifs - 1)
		if wrapper.Parent then wrapper:Destroy() end
	end)
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
		BackgroundColor3 = COLORS.Card, BorderSizePixel = 0,
		LayoutOrder = nextOrder(parent), Parent = parent,
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

local function AddToggle(sec, title, desc, default, callback)
	local h = desc and 44 or 34
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, h), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(1, -84, 0, desc and 20 or h), Position = UDim2.new(0, 14, 0, desc and 4 or 0),
		Text = title, Parent = row})
	if desc then
		label({Size = UDim2.new(1, -84, 0, 16), Position = UDim2.new(0, 14, 0, 23), Text = desc,
			TextSize = 10, TextColor3 = COLORS.TextDark, TextTruncate = Enum.TextTruncate.AtEnd, Parent = row})
	end
	local switch = create("TextButton", {
		Size = UDim2.fromOffset(42, 22), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
		BackgroundColor3 = default and COLORS.Accent or COLORS.SwitchOff, Text = "", AutoButtonColor = false, Parent = row,
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
			safeCall(callback, state)
			Notify(title, state and "Enabled" or "Disabled", 1.5)
		end
	end
	function control.Get() return state end
	switch.MouseButton1Click:Connect(function() control.Set(not state) end)

	registerRow(sec, row, title)
	return control
end

local function AddSlider(sec, title, minVal, maxVal, default, step, unit, callback)
	local decimals = step >= 1 and 0 or (step >= 0.1 and 1 or 2)
	local function fmt(v) return string.format("%." .. decimals .. "f", v) .. unit end

	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 52), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(1, -90, 0, 20), Position = UDim2.new(0, 14, 0, 4), Text = title, Parent = row})

	local valBox = create("Frame", {Size = UDim2.fromOffset(56, 20), AnchorPoint = Vector2.new(1, 0),
		Position = UDim2.new(1, -14, 0, 4), BackgroundColor3 = COLORS.CardHeader, Parent = row})
	corner(valBox, 6)
	local valLabel = label({Size = UDim2.fromScale(1, 1), Text = fmt(default), TextSize = 10,
		Font = Enum.Font.GothamBold, TextXAlignment = Enum.TextXAlignment.Center, Parent = valBox})

	local hit = create("Frame", {Size = UDim2.new(1, -28, 0, 22), Position = UDim2.new(0, 14, 0, 28),
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
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 40), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	local base = danger and COLORS.Danger or COLORS.CardHeader
	local over = danger and Color3.fromRGB(225, 80, 100) or COLORS.CardHover
	local btn = create("TextButton", {
		Size = UDim2.new(1, -28, 0, 32), Position = UDim2.new(0, 14, 0.5, -16), BackgroundColor3 = base,
		Text = title, TextColor3 = COLORS.TextMain, TextSize = 12, Font = Enum.Font.GothamBold,
		AutoButtonColor = false, Parent = row,
	})
	corner(btn, 8)
	stroke(btn, danger and COLORS.Danger or COLORS.Stroke, 1)
	btn.MouseEnter:Connect(function() tween(btn, 0.15, {BackgroundColor3 = over}) end)
	btn.MouseLeave:Connect(function() tween(btn, 0.15, {BackgroundColor3 = base}) end)
	btn.MouseButton1Down:Connect(function() tween(btn, 0.08, {Size = UDim2.new(1, -34, 0, 30)}) end)
	btn.MouseButton1Up:Connect(function() tween(btn, 0.12, {Size = UDim2.new(1, -28, 0, 32)}) end)
	btn.MouseButton1Click:Connect(function() safeCall(callback) end)

	registerRow(sec, row, title)
	return btn
end

-- Campo de texto (confirma ao sair do campo / Enter)
local function AddInput(sec, title, placeholder, default, callback)
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 38), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(1, -190, 1, 0), Position = UDim2.new(0, 14, 0, 0), Text = title, Parent = row})
	local boxFrame = create("Frame", {
		Size = UDim2.fromOffset(160, 26), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
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
		safeCall(callback, box.Text)
	end)
	registerRow(sec, row, title)
	return box
end

local function AddInfo(sec, title, value)
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 26), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(sec.Frame), Parent = sec.Frame})
	label({Size = UDim2.new(0.4, 0, 1, 0), Position = UDim2.new(0, 14, 0, 0), Text = title,
		TextSize = 11, TextColor3 = COLORS.TextDark, Parent = row})
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
-- 8. BUSCA (com debounce; navega até o resultado)
-- ==========================================
local searchToken = 0
local function applySearch()
	local q = (string.lower(SearchBox.Text):gsub("^%s*(.-)%s*$", "%1"))
	local firstMatchSec, currentHasMatch, anyMatch = nil, false, false

	for _, sec in ipairs(sections) do
		local any = false
		for _, r in ipairs(sec.Rows) do
			local match = (q == "") or (string.find(r.Title, q, 1, true) ~= nil)
			if r.Frame.Visible ~= match then r.Frame.Visible = match end
			if match then any = true end
		end
		local secVisible = (q == "") or any
		if sec.Frame.Visible ~= secVisible then sec.Frame.Visible = secVisible end
		if q ~= "" and any then
			anyMatch = true
			firstMatchSec = firstMatchSec or sec
			if sec.Tab == activeTab then
				local g = groups[sec.Tab]
				if sec.Sub == nil or (g and g.Active == sec.Sub) then currentHasMatch = true end
			end
		end
	end

	NoResults.Visible = (q ~= "" and not anyMatch)
	if q ~= "" and firstMatchSec and not currentHasMatch then
		setActiveTab(firstMatchSec.Tab)
		if firstMatchSec.Sub and groups[firstMatchSec.Tab] then groups[firstMatchSec.Tab].Set(firstMatchSec.Sub) end
	end
end
SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
	searchToken = searchToken + 1
	local token = searchToken
	task.delay(0.12, function()
		if token == searchToken then applySearch() end
	end)
end)

-- ==========================================
-- 9. ABAS PRINCIPAIS
-- ==========================================

-- ---------- 9.1 HOME ----------
local WelcomeCard = create("Frame", {
	Size = UDim2.new(1, -8, 0, 72), BackgroundColor3 = COLORS.Card, BorderSizePixel = 0,
	LayoutOrder = nextOrder(HomeScroll), Parent = HomeScroll,
})
corner(WelcomeCard, 12)
stroke(WelcomeCard, COLORS.Stroke, 1)
local HomeAvatar = create("ImageLabel", {
	Size = UDim2.fromOffset(48, 48), Position = UDim2.new(0, 14, 0.5, -24),
	BackgroundColor3 = COLORS.CardHeader, Image = "", Parent = WelcomeCard,
})
corner(HomeAvatar, 24)
stroke(HomeAvatar, COLORS.Accent, 1.5, 0.2)
label({Size = UDim2.new(1, -90, 0, 14), Position = UDim2.new(0, 74, 0, 14), Text = "Welcome back,",
	TextSize = 11, TextColor3 = COLORS.TextDark, Parent = WelcomeCard})
label({Size = UDim2.new(1, -90, 0, 20), Position = UDim2.new(0, 74, 0, 32),
	Text = LocalPlayer.DisplayName .. " (@" .. LocalPlayer.Name .. ")", Font = Enum.Font.GothamBold,
	TextSize = 14, TextTruncate = Enum.TextTruncate.AtEnd, Parent = WelcomeCard})

task.spawn(function()
	local ok, img = pcall(function()
		return (Players:GetUserThumbnailAsync(LocalPlayer.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100))
	end)
	if ok and img then
		AvatarImg.Image = img
		HomeAvatar.Image = img
	end
end)

local StatsGrid = create("Frame", {
	Size = UDim2.new(1, -8, 0, 54), BackgroundTransparency = 1,
	LayoutOrder = nextOrder(HomeScroll), Parent = HomeScroll,
})
create("UIGridLayout", {CellSize = UDim2.new(0.235, 0, 1, 0), CellPadding = UDim2.new(0.02, 0, 0, 0),
	SortOrder = Enum.SortOrder.LayoutOrder, Parent = StatsGrid})

local function createStatBox(title, initial)
	local box = create("Frame", {BackgroundColor3 = COLORS.Card, BorderSizePixel = 0,
		LayoutOrder = nextOrder(StatsGrid), Parent = StatsGrid})
	corner(box, 10)
	stroke(box, COLORS.Stroke, 1)
	label({Size = UDim2.new(1, -16, 0, 16), Position = UDim2.new(0, 10, 0, 8), Text = title,
		TextSize = 10, TextColor3 = COLORS.TextDark, Parent = box})
	return label({Size = UDim2.new(1, -16, 0, 20), Position = UDim2.new(0, 10, 0, 26), Text = initial,
		TextSize = 13, Font = Enum.Font.GothamBold, Parent = box})
end

local PlayersVal = createStatBox("Players", #Players:GetPlayers() .. "/" .. Players.MaxPlayers)
local SessionVal = createStatBox("Session", "00m 00s")
local FpsVal     = createStatBox("FPS", "--")
local PingVal    = createStatBox("Ping", "--")

local GameSec = AddSection(HomeScroll, "Game")
local GameNameVal = AddInfo(GameSec, "Name", "Loading...")
AddInfo(GameSec, "Place ID", tostring(game.PlaceId))
local AgeVal = AddInfo(GameSec, "Server age", "--")
local GameScriptVal = AddInfo(GameSec, "Game script", "None for this game")

local loadGameByName -- forward declaration (seção 10)
local ScriptsSec = AddSection(HomeScroll, "Game Scripts")
AddText(ScriptsSec, "Scripts de jogo carregam sozinhos quando você entra num jogo suportado. Se a detecção falhar, force manualmente:")
AddButton(ScriptsSec, "Load: Merge a Mini Army", function() loadGameByName("Merge a Mini Army") end)

local AboutSec = AddSection(HomeScroll, "About")
AddText(AboutSec, "Dark Shadow Hub v3.0\nAbra/feche o menu pelo botão flutuante ou pela tecla (padrão: Right Shift). Arraste a sidebar ou o cabeçalho para mover a janela.")

-- ---------- 9.2 PLAYER ----------
local function restoreSpeed()
	local hum = getHum()
	if hum and Original.Speed then hum.WalkSpeed = Original.Speed end
	Original.Speed = nil
end

local function restoreJump()
	local hum = getHum()
	if hum and Original.Jump then
		if Original.JumpIsPower then hum.JumpPower = Original.Jump else hum.JumpHeight = Original.Jump end
	end
	Original.Jump = nil
end

local function rebuildNoclip()
	table.clear(noclipParts)
	local c = LocalPlayer.Character
	if not c then return end
	for _, p in ipairs(c:GetDescendants()) do
		if p:IsA("BasePart") then noclipParts[#noclipParts + 1] = p end
	end
end

local function restoreNoclip()
	for part in pairs(noclipTouched) do
		if part.Parent then part.CanCollide = true end
	end
	noclipTouched = {}
	table.clear(noclipParts)
end

local function stopFly()
	if flyObjs then
		for _, o in pairs(flyObjs) do if o then o:Destroy() end end
		flyObjs = nil
	end
	flyVelocity = Vector3.zero
	local hum = getHum()
	if hum then
		hum.PlatformStand = false
		pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
	end
end

local function startFly()
	local root, hum = getRoot(), getHum()
	if not root or not hum then return end
	stopFly()
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
	flyObjs = {bv = bv, bg = bg}
end

local function getMoveVector()
	if controlsModule then
		local ok, v = pcall(function() return controlsModule:GetMoveVector() end)
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

local MoveSec = AddSection(PlayerScroll, "Movement", "Player")
AddToggle(MoveSec, "Custom WalkSpeed", "Overrides your walk speed", false, function(on)
	State.SpeedOn = on
	if not on then restoreSpeed() end
end)
AddSlider(MoveSec, "WalkSpeed", 16, 200, State.Speed, 1, "", function(v) State.Speed = v end)
AddToggle(MoveSec, "Custom JumpPower", "Overrides your jump strength", false, function(on)
	State.JumpOn = on
	if not on then restoreJump() end
end)
AddSlider(MoveSec, "JumpPower", 50, 300, State.Jump, 1, "", function(v) State.Jump = v end)
AddToggle(MoveSec, "Infinite Jump", "Jump again while in the air", false, function(on) State.InfJump = on end)
AddToggle(MoveSec, "Noclip", "Walk through walls", false, function(on)
	State.Noclip = on
	if on then rebuildNoclip() else restoreNoclip() end
end)

local FlySec = AddSection(PlayerScroll, "Fly", "Player")
AddToggle(FlySec, "Fly", "Space / Ctrl = up / down. Mobile: look direction", false, function(on)
	State.Fly = on
	if on then startFly() else stopFly() end
end)
AddSlider(FlySec, "Fly Speed", 10, 250, State.FlySpeed, 1, "", function(v) State.FlySpeed = v end)

local CharSec = AddSection(PlayerScroll, "Character", "Player")
AddButton(CharSec, "Reset Character", function()
	local hum = getHum()
	if hum then hum.Health = 0 end
end)
AddButton(CharSec, "Copy Position", function()
	local root = getRoot()
	if not root then Notify("Position", "Character not found", 2) return end
	local p = root.Position
	local text = string.format("%.1f, %.1f, %.1f", p.X, p.Y, p.Z)
	if clipboardFn then clipboardFn(text); Notify("Position", "Copied: " .. text, 2.5)
	else Notify("Position", text, 4) end
end)

-- ---------- 9.3 MAIN > VISUALS ----------
local fbOrig, fogOrig
local function restoreFullbright()
	if fbOrig then
		for k, v in pairs(fbOrig) do pcall(function() Lighting[k] = v end) end
		fbOrig = nil
	end
end
local function restoreFog()
	if fogOrig then
		pcall(function() Lighting.FogEnd = fogOrig.FogEnd; Lighting.FogStart = fogOrig.FogStart end)
		for atm, d in pairs(fogOrig.Atmos) do if atm.Parent then atm.Density = d end end
		fogOrig = nil
	end
end

local VisSec = AddSection(VisualsScroll, "Lighting", "Main", "Visuals")
AddToggle(VisSec, "Fullbright", "Removes darkness and shadows", false, function(on)
	State.Fullbright = on
	if on then
		fbOrig = fbOrig or {Brightness = Lighting.Brightness, ClockTime = Lighting.ClockTime,
			GlobalShadows = Lighting.GlobalShadows, Ambient = Lighting.Ambient, OutdoorAmbient = Lighting.OutdoorAmbient}
	else
		restoreFullbright()
	end
end)
AddToggle(VisSec, "Remove Fog", "Clears fog and atmosphere haze", false, function(on)
	State.NoFog = on
	if on then
		if not fogOrig then
			fogOrig = {FogEnd = Lighting.FogEnd, FogStart = Lighting.FogStart, Atmos = {}}
			for _, c in ipairs(Lighting:GetChildren()) do
				if c:IsA("Atmosphere") then fogOrig.Atmos[c] = c.Density end
			end
		end
	else
		restoreFog()
	end
end)

local CamSec = AddSection(VisualsScroll, "Camera", "Main", "Visuals")
AddToggle(CamSec, "Custom FOV", "Overrides the field of view", false, function(on)
	State.FovOn = on
	local cam = workspace.CurrentCamera
	if on then
		if cam then Original.Fov = Original.Fov or cam.FieldOfView end
	else
		if cam and Original.Fov then cam.FieldOfView = Original.Fov end
		Original.Fov = nil
	end
end)
AddSlider(CamSec, "Field of View", 30, 120, State.Fov, 1, "°", function(v) State.Fov = v end)
AddSlider(CamSec, "Max Zoom Distance", 10, 1000, math.clamp(math.floor(origMaxZoom), 10, 1000), 1, "", function(v)
	LocalPlayer.CameraMaxZoomDistance = v
end)

-- ---------- 9.4 MAIN > UTILITY ----------
local afkConn, rejoinConn, lowGfxOrig
local antiAfkToggle, lowGfxToggle, noRenderToggle

local function setAntiAfk(on)
	State.AntiAfk = on
	if on then
		if not afkConn then
			afkConn = LocalPlayer.Idled:Connect(function()
				pcall(function()
					VirtualUser:CaptureController()
					VirtualUser:ClickButton2(Vector2.new())
				end)
			end)
		end
	elseif afkConn then
		afkConn:Disconnect()
		afkConn = nil
	end
end

local function restoreLowGfx()
	if lowGfxOrig then
		if lowGfxOrig.Quality then pcall(function() settings().Rendering.QualityLevel = lowGfxOrig.Quality end) end
		if lowGfxOrig.Decoration ~= nil then pcall(function() workspace.Terrain.Decoration = lowGfxOrig.Decoration end) end
		lowGfxOrig = nil
	end
end

local function setLowGfx(on)
	State.LowGfx = on
	if on then
		if not lowGfxOrig then
			lowGfxOrig = {}
			pcall(function() lowGfxOrig.Quality = settings().Rendering.QualityLevel end)
			pcall(function() lowGfxOrig.Decoration = workspace.Terrain.Decoration end)
		end
		pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
		pcall(function() workspace.Terrain.Decoration = false end)
	else
		restoreLowGfx()
	end
end

local function setNoRender(on)
	State.NoRender = on
	pcall(function() RunService:Set3dRenderingEnabled(not on) end)
end

local rejoining = false
local function setAutoRejoin(on)
	State.AutoRejoin = on
	if on then
		if not rejoinConn then
			rejoinConn = GuiService.ErrorMessageChanged:Connect(function(msg)
				if State.AutoRejoin and not rejoining and msg and msg ~= "" then
					rejoining = true
					task.wait(3)
					pcall(function() TeleportService:Teleport(game.PlaceId, LocalPlayer) end)
					task.delay(10, function() rejoining = false end)
				end
			end)
		end
	elseif rejoinConn then
		rejoinConn:Disconnect()
		rejoinConn = nil
	end
end

local UtilSec = AddSection(UtilityScroll, "Utility", "Main", "Utility")
antiAfkToggle  = AddToggle(UtilSec, "Anti-AFK", "Prevents the idle kick", false, setAntiAfk)
lowGfxToggle   = AddToggle(UtilSec, "Low Graphics", "Lower quality level for more FPS", false, setLowGfx)
noRenderToggle = AddToggle(UtilSec, "Disable 3D Rendering", "Black screen, huge FPS/CPU saving while AFK", false, setNoRender)
AddToggle(UtilSec, "Auto Rejoin", "Rejoins automatically if you get disconnected", false, setAutoRejoin)
if setfpscap then
	AddSlider(UtilSec, "FPS Cap", 30, 240, 60, 5, "", function(v) pcall(setfpscap, v) end)
end

-- ---------- 9.5 MAIN > SERVER ----------
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
			return HttpService:JSONDecode(game:HttpGet(url))
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

local SrvInfo = AddSection(ServerScroll, "Server Info", "Main", "Server")
local SrvPlayersVal = AddInfo(SrvInfo, "Players", #Players:GetPlayers() .. "/" .. Players.MaxPlayers)
AddInfo(SrvInfo, "Place ID", tostring(game.PlaceId))
AddInfo(SrvInfo, "Job ID", game.JobId ~= "" and game.JobId or "Studio / Private")
local SrvAgeVal = AddInfo(SrvInfo, "Server age", "--")

local SrvActions = AddSection(ServerScroll, "Actions", "Main", "Server")
AddButton(SrvActions, "Copy Job ID", function() copyText(game.JobId, "Job ID") end)
AddButton(SrvActions, "Copy Place ID", function() copyText(tostring(game.PlaceId), "Place ID") end)
AddButton(SrvActions, "Rejoin Server", rejoin)
AddButton(SrvActions, "Server Hop", serverHop)

-- ---------- 9.6 SETTINGS ----------
local UnloadHub -- forward declaration
local keyBtnRef

local UISec = AddSection(SettingsScroll, "Interface", "Settings")
do
	local row = create("Frame", {Size = UDim2.new(1, 0, 0, 38), BackgroundTransparency = 1,
		LayoutOrder = nextOrder(UISec.Frame), Parent = UISec.Frame})
	label({Size = UDim2.new(1, -130, 1, 0), Position = UDim2.new(0, 14, 0, 0), Text = "Menu Keybind", Parent = row})
	keyBtnRef = create("TextButton", {
		Size = UDim2.fromOffset(96, 26), AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -14, 0.5, 0),
		BackgroundColor3 = COLORS.CardHeader, Text = toggleKey.Name, TextColor3 = COLORS.TextMain, TextSize = 11,
		Font = Enum.Font.GothamBold, AutoButtonColor = false, Parent = row,
	})
	corner(keyBtnRef, 6)
	stroke(keyBtnRef, COLORS.Stroke, 1)
	keyBtnRef.MouseButton1Click:Connect(function()
		listeningKey = true
		keyBtnRef.Text = "Press a key..."
		tween(keyBtnRef, 0.15, {BackgroundColor3 = COLORS.AccentDark})
	end)
	registerRow(UISec, row, "Menu Keybind")
end

AddSlider(UISec, "UI Scale", 0.7, 1.3, 1, 0.05, "x", function(v)
	userScale = v
	currentScale = computeScale()
	HubScale.Scale = currentScale
	hubCenter = clampCenter(hubCenter)
	MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
end)
AddToggle(UISec, "Notifications", "Show pop-up messages", true, function(on) notificationsEnabled = on end)
AddToggle(UISec, "Snow Effect", "Disable for better performance", true, function(on)
	State.Snow = on
	SnowContainer.Visible = on and isOpen
end)

local DangerSec = AddSection(SettingsScroll, "Danger Zone", "Settings")
AddButton(DangerSec, "Unload Hub", function() UnloadHub() end, true)

-- ================================================================= --
-- 10. SISTEMA DE SCRIPTS POR JOGO
--  Cada jogo vira uma ABA própria na sidebar (só no jogo dele).
--  Para adicionar outro jogo: copie o bloco RegisterGame({...}).
-- ================================================================= --
local cleanupFns = {}
local GameModules = {}

local function RegisterGame(def)
	table.insert(GameModules, def)
end

local function buildGameContext(def, group)
	return {
		-- cria uma sub-aba e devolve um objeto com :Section(titulo)
		Page = function(name, icon)
			local scroll = group.Add(name, icon or ICONS.Merge)
			return {Section = function(title) return AddSection(scroll, title, def.TabName, name) end}
		end,
		AddToggle = AddToggle, AddSlider = AddSlider, AddButton = AddButton,
		AddInput = AddInput, AddInfo = AddInfo, AddText = AddText,
		Notify = Notify, track = track, safeCall = safeCall,
		getRoot = getRoot, getHum = getHum, copy = clipboardFn,
		ScreenGui = ScreenGui, Icons = ICONS,
		IsVisible = function() return isOpen and tabs[def.TabName] ~= nil and tabs[def.TabName].Active end,
		SetAntiAfk = function(on) setAntiAfk(on); antiAfkToggle.Set(on, true) end,
		SetLowGfx = function(on) setLowGfx(on); lowGfxToggle.Set(on, true) end,
		SetNoRender = function(on) setNoRender(on); noRenderToggle.Set(on, true) end,
		OnUnload = function(fn) table.insert(cleanupFns, fn) end,
	}
end

local function matchesGame(def, gameName)
	for _, id in ipairs(def.PlaceIds or {}) do
		if id == game.PlaceId then return true end
	end
	for _, id in ipairs(def.GameIds or {}) do
		if id == game.GameId then return true end
	end
	if gameName and def.NameMatch then
		return string.find(string.lower(gameName), string.lower(def.NameMatch), 1, true) ~= nil
	end
	return false
end

local loadedGames = {}
local function loadGame(def)
	if loadedGames[def.Name] or not ScreenGui.Parent then return false end
	loadedGames[def.Name] = true
	local page = createTab(def.TabName, def.Icon or ICONS.Merge, false, ORDER.Game)
	local group = createSubGroup(def.TabName, page)
	local ok, err = pcall(def.Build, buildGameContext(def, group))
	if ok then
		GameScriptVal.Text = def.Name
		Notify("Game Script", def.Name .. " loaded", 3)
	else
		warn("[ShadowHub] " .. def.Name .. ": " .. tostring(err))
		GameScriptVal.Text = def.Name .. " (error)"
		Notify("Game Script", "Failed to load " .. def.Name, 4)
	end
	return ok
end

loadGameByName = function(name)
	for _, def in ipairs(GameModules) do
		if def.Name == name then
			if loadedGames[name] then
				Notify("Game Script", name .. " is already loaded", 2.5)
				setActiveTab(def.TabName)
			elseif loadGame(def) then
				setActiveTab(def.TabName)
			end
			return
		end
	end
end

-- ================================================================= --
-- 11. JOGO: MERGE A MINI ARMY
--  Auto Merge / Buy / Collect / Upgrade / Spin / Equip / Rebirth,
--  Auto Capture de territórios, modo AFK e ferramentas de diagnóstico.
--  Funciona detectando e acionando os botões da interface do jogo.
-- ================================================================= --
RegisterGame({
	Name = "Merge a Mini Army",
	TabName = "Mini Army",
	Icon = ICONS.Merge,
	PlaceIds = {},                       -- opcional: coloque o PlaceId para detecção exata
	NameMatch = "merge a mini army",
	Build = function(ctx)
		local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

		-- ---------- Configuração ----------
		local cfg = {
			Merge = false, Buy = false, Collect = false, Upgrade = false,
			Spin = false, Equip = false, Battle = false, Rebirth = false, Custom = false,
			Delay = 0.6, RebirthEvery = 20, CustomWords = {},
			AutoCapture = false, Range = 25, Instant = false,
			AfkRebirth = false,
		}
		local running = true
		local stats = {clicks = 0, rebirths = 0}
		local ctl = {}

		-- Categorias de botões (palavras buscadas no texto/nome do botão)
		local CATS = {
			{key = "Merge",   words = {"merge", "combine"}, boostOk = true},
			{key = "Rebirth", words = {"rebirth"}, boostOk = true},
			{key = "Equip",   words = {"equip best", "equip all", "best units"}},
			{key = "Collect", words = {"collect", "claim"}},
			{key = "Upgrade", words = {"upgrade"}},
			{key = "Spin",    words = {"spin"}},
			{key = "Battle",  words = {"start battle", "next wave", "next stage", "start wave", "fight"}},
			{key = "Buy",     words = {"buy", "spawn", "summon", "recruit", "hire", "deploy"}},
		}
		-- Nunca clica em nada ligado a Robux / gamepass / presentes
		local BLOCK_HARD = {"robux", "r$", "gamepass", "game pass", "premium", "gift", "donat", "purchase", "vip"}
		-- Boosts (x2, 2x...) só passam em Merge/Rebirth
		local BOOST = {"x2", "2x", "x3", "3x", "x4", "4x"}
		local CONFIRM = {"confirm", "yes"}

		local function hasAny(text, words)
			for _, w in ipairs(words) do
				if string.find(text, w, 1, true) then return true end
			end
			return false
		end

		-- ---------- Rastreio de botões (sob demanda) ----------
		local buttons = {}
		local trackingButtons = false

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
			end)
			ctx.track(PlayerGui.DescendantAdded:Connect(addButton))
			ctx.track(PlayerGui.DescendantRemoving:Connect(function(d) buttons[d] = nil end))
		end

		-- Texto/categoria em cache (recalcula a cada 2s, não a cada ciclo)
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
			info.confirm = hasAny(t, CONFIRM)
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

		-- Dispara os eventos do botão SEM duplicar o clique
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

		local function clickButton(b)
			if fireSignal(b.MouseButton1Click) then return true end
			if fireSignal(b.Activated) then return true end
			if firesignal then
				if pcall(firesignal, b.MouseButton1Click) then return true end
				if pcall(firesignal, b.Activated) then return true end
			end
			return false
		end

		local AUTO_KEYS = {"Merge", "Buy", "Collect", "Upgrade", "Spin", "Equip", "Battle", "Rebirth", "Custom"}
		local function anyAutoOn()
			for _, k in ipairs(AUTO_KEYS) do
				if cfg[k] then return true end
			end
			return false
		end

		local lastRebirth, confirmUntil = 0, 0

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(cfg.Delay)
				if anyAutoOn() then
					ensureButtonTracking()
					local now = os.clock()
					local list = {}
					for b in pairs(buttons) do list[#list + 1] = b end

					for i = 1, #list do
						local b = list[i]
						local info = buttons[b]
						if info and b.Parent then
							refreshInfo(b, info, now)
							if not info.block then
								local doClick = false
								local isRebirth = false
								local cat = info.cat

								if cat and cfg[cat.key] and (not info.boost or cat.boostOk) then
									if cat.key == "Rebirth" then
										if now - lastRebirth >= cfg.RebirthEvery then doClick, isRebirth = true, true end
									else
										doClick = true
									end
								end
								if not doClick and cfg.Custom and #cfg.CustomWords > 0 and hasAny(info.text, cfg.CustomWords) then
									doClick = true
								end
								-- Confirma o diálogo logo depois de um rebirth
								if not doClick and cfg.Rebirth and now < confirmUntil and info.confirm then
									doClick = true
								end

								if doClick and isShown(b) then
									if clickButton(b) then
										stats.clicks = stats.clicks + 1
										if isRebirth then
											stats.rebirths = stats.rebirths + 1
											lastRebirth = now
											confirmUntil = now + 4
										end
									end
								end
							end
						end
						if i % 60 == 0 then task.wait() end
					end
				end
			end
		end)

		-- ---------- Territórios (ProximityPrompts, sob demanda) ----------
		local KEYWORDS = {"captur", "conquer", "garrison", "compound", "airbase", "territor"}
		local prompts, originalHold = {}, {}
		local territoryCache = setmetatable({}, {__mode = "k"})
		local trackingPrompts = false

		local function promptPos(p)
			local par = p.Parent
			if not par then return nil end
			if par:IsA("BasePart") then return par.Position end
			if par:IsA("Attachment") then return par.WorldPosition end
			if par:IsA("Model") then return par:GetPivot().Position end
			return nil
		end

		local function isTerritory(p)
			local cached = territoryCache[p]
			if cached ~= nil then return cached end
			local par = p.Parent
			if not par then return false end
			local text = string.lower(p.ActionText .. " " .. p.ObjectText .. " " .. par.Name .. " " .. (par.Parent and par.Parent.Name or ""))
			local result = hasAny(text, KEYWORDS)
			territoryCache[p] = result
			return result
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

		local function nearestTerritory(minDist)
			local root = ctx.getRoot()
			if not root then return nil end
			local best, bestD
			for p in pairs(prompts) do
				if p.Parent and isTerritory(p) then
					local pos = promptPos(p)
					if pos then
						local d = (pos - root.Position).Magnitude
						if d > minDist and (not bestD or d < bestD) then best, bestD = pos, d end
					end
				end
			end
			return best
		end

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(0.5)
				if cfg.AutoCapture and fireproximityprompt then
					local root = ctx.getRoot()
					if root then
						local rpos = root.Position
						for p in pairs(prompts) do
							if p.Parent and p.Enabled and isTerritory(p) then
								local pos = promptPos(p)
								if pos and (pos - rpos).Magnitude <= cfg.Range then
									pcall(fireproximityprompt, p)
								end
							end
						end
					end
				end
			end
		end)

		-- ================= UI: sub-aba AUTO FARM =================
		local farm = ctx.Page("Auto Farm", ctx.Icons.Merge)

		local afkSec = farm.Section("AFK Mode")
		local AFK_KEYS = {"Merge", "Buy", "Collect", "Upgrade", "Spin", "Equip"}
		ctl.AFK = ctx.AddToggle(afkSec, "AFK Farm Mode", "Anti-AFK + auto merge, buy, collect, upgrade, capture", false, function(on)
			ctx.SetAntiAfk(on)
			for _, key in ipairs(AFK_KEYS) do
				cfg[key] = on
				ctl[key].Set(on, true)
			end
			cfg.Rebirth = on and cfg.AfkRebirth
			ctl.Rebirth.Set(cfg.Rebirth, true)
			local cap = on and fireproximityprompt ~= nil
			cfg.AutoCapture = cap
			ctl.AutoCapture.Set(cap, true)
			if cap then ensurePromptTracking() end
			if on then ensureButtonTracking() end
			if on and not (firesignal or getconnections) then
				ctx.Notify("AFK Mode", "Executor can't click UI buttons", 4)
			end
		end)
		ctx.AddToggle(afkSec, "AFK includes Rebirth", "Also auto rebirths while AFK Mode is on", false, function(on)
			cfg.AfkRebirth = on
			if ctl.AFK.Get() then
				cfg.Rebirth = on
				ctl.Rebirth.Set(on, true)
			end
		end)
		ctx.AddToggle(afkSec, "AFK Low Graphics", "Reduces FPS usage while idle", false, function(on) ctx.SetLowGfx(on) end)
		ctx.AddToggle(afkSec, "AFK Disable 3D Rendering", "Black screen, minimum CPU/GPU usage", false, function(on) ctx.SetNoRender(on) end)

		local autoSec = farm.Section("Automation")
		ctl.Merge   = ctx.AddToggle(autoSec, "Auto Merge", "Clicks the Merge button when it is visible", false, function(on) cfg.Merge = on; if on then ensureButtonTracking() end end)
		ctl.Buy     = ctx.AddToggle(autoSec, "Auto Buy / Spawn Units", "Clicks buy / spawn / summon buttons", false, function(on) cfg.Buy = on; if on then ensureButtonTracking() end end)
		ctl.Collect = ctx.AddToggle(autoSec, "Auto Collect / Claim", "Clicks collect and claim buttons", false, function(on) cfg.Collect = on; if on then ensureButtonTracking() end end)
		ctl.Upgrade = ctx.AddToggle(autoSec, "Auto Upgrade", "Clicks upgrade buttons", false, function(on) cfg.Upgrade = on; if on then ensureButtonTracking() end end)
		ctl.Spin    = ctx.AddToggle(autoSec, "Auto Spin", "Clicks free spin buttons (Robux ones are skipped)", false, function(on) cfg.Spin = on; if on then ensureButtonTracking() end end)
		ctl.Equip   = ctx.AddToggle(autoSec, "Auto Equip Best", "Clicks equip best / equip all buttons", false, function(on) cfg.Equip = on; if on then ensureButtonTracking() end end)
		ctl.Battle  = ctx.AddToggle(autoSec, "Auto Start Battle / Next Wave", "Clicks start battle and next wave buttons", false, function(on) cfg.Battle = on; if on then ensureButtonTracking() end end)
		ctx.AddSlider(autoSec, "Action Delay", 0.2, 3, cfg.Delay, 0.1, "s", function(v) cfg.Delay = v end)
		ctx.AddText(autoSec, "Botões ligados a Robux, gamepass, VIP ou presentes são sempre ignorados.")

		local rebSec = farm.Section("Rebirth")
		ctl.Rebirth = ctx.AddToggle(rebSec, "Auto Rebirth", "Careful: resets your progress", false, function(on)
			cfg.Rebirth = on
			if on then ensureButtonTracking() end
		end)
		ctx.AddSlider(rebSec, "Rebirth Cooldown", 5, 300, cfg.RebirthEvery, 5, "s", function(v) cfg.RebirthEvery = v end)
		ctx.AddText(rebSec, "O cooldown evita rebirth em sequência. Após clicar em Rebirth o script também confirma o diálogo (Confirm / Yes).")

		-- ================= UI: sub-aba TERRITORIES =================
		local terr = ctx.Page("Territories", ctx.Icons.Combat)
		local capSec = terr.Section("Capture")
		ctl.AutoCapture = ctx.AddToggle(capSec, "Auto Capture", "Triggers capture prompts of nearby territories", false, function(on)
			if on and not fireproximityprompt then
				ctx.Notify("Auto Capture", "Your executor lacks fireproximityprompt", 4)
				ctl.AutoCapture.Set(false, true)
				return
			end
			cfg.AutoCapture = on
			if on then ensurePromptTracking() end
		end)
		ctx.AddSlider(capSec, "Capture Range", 5, 60, cfg.Range, 1, " st", function(v) cfg.Range = v end)
		ctx.AddToggle(capSec, "Instant Interact", "Removes the hold time on prompts", false, function(on)
			cfg.Instant = on
			if on then
				ensurePromptTracking()
				for p in pairs(prompts) do applyInstant(p) end
			else
				restoreInstant()
			end
		end)
		ctx.AddButton(capSec, "Teleport to Next Territory", function()
			ensurePromptTracking()
			local root = ctx.getRoot()
			local pos = nearestTerritory(15)
			if not root then ctx.Notify("Teleport", "Character not found", 2) return end
			if not pos then ctx.Notify("Teleport", "No territory found yet", 3) return end
			root.CFrame = CFrame.new(pos + Vector3.new(0, 4, 0))
			ctx.Notify("Teleport", "Moved to the nearest territory", 2)
		end)

		-- ================= UI: sub-aba TOOLS =================
		local tools = ctx.Page("Tools", ctx.Icons.Shield)

		local customSec = tools.Section("Custom Click")
		ctx.AddToggle(customSec, "Custom Auto Click", "Clicks any button containing your keywords", false, function(on)
			cfg.Custom = on
			if on then ensureButtonTracking() end
		end)
		ctx.AddInput(customSec, "Keywords", "ex: claim, daily", "", function(text)
			local words = {}
			for w in string.gmatch(string.lower(text), "[^,]+") do
				w = w:gsub("^%s*(.-)%s*$", "%1")
				if #w > 0 then words[#words + 1] = w end
			end
			cfg.CustomWords = words
			ctx.Notify("Custom Click", #words .. " keyword(s) set", 2)
		end)
		ctx.AddText(customSec, "Separe palavras por vírgula. Útil se o jogo usar nomes diferentes nos botões.")

		local infoSec = tools.Section("Status")
		local clicksVal  = ctx.AddInfo(infoSec, "Buttons clicked", "0")
		local rebirthVal = ctx.AddInfo(infoSec, "Rebirth clicks", "0")
		local btnVal     = ctx.AddInfo(infoSec, "Buttons tracked", "0")
		local trackedVal = ctx.AddInfo(infoSec, "Prompts tracked", "0")
		local nearVal    = ctx.AddInfo(infoSec, "Territories in range", "0")

		local devSec = tools.Section("Developer Tools")
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
		ctx.AddButton(devSec, "Scan Buttons", function()
			ensureButtonTracking()
			local out, now = {}, os.clock()
			for b, info in pairs(buttons) do
				if b.Parent and isShown(b) then
					refreshInfo(b, info, now)
					table.insert(out, string.format("%s | %s", b:GetFullName(), info.text))
				end
			end
			dump(out, "Scan Buttons")
		end)
		ctx.AddButton(devSec, "Scan Remotes", function()
			local out = {}
			for _, d in ipairs(game:GetService("ReplicatedStorage"):GetDescendants()) do
				if d:IsA("RemoteEvent") or d:IsA("RemoteFunction") then
					table.insert(out, d.ClassName .. ": " .. d:GetFullName())
				end
			end
			dump(out, "Scan Remotes")
		end)
		ctx.AddText(devSec, "Se algum auto não clicar: abra o menu do jogo (botão visível), rode Scan Buttons e ajuste as palavras em Custom Click.")

		-- Atualiza o status 1x/s, só com a aba visível
		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(1)
				if ctx.IsVisible() then
					local total, near = 0, 0
					local root = ctx.getRoot()
					for p in pairs(prompts) do
						total = total + 1
						if root and p.Parent and isTerritory(p) then
							local pos = promptPos(p)
							if pos and (pos - root.Position).Magnitude <= cfg.Range then near = near + 1 end
						end
					end
					local bc = 0
					for _ in pairs(buttons) do bc = bc + 1 end
					clicksVal.Text = tostring(stats.clicks)
					rebirthVal.Text = tostring(stats.rebirths)
					btnVal.Text = tostring(bc)
					trackedVal.Text = tostring(total)
					nearVal.Text = tostring(near)
				end
			end
		end)

		ctx.OnUnload(function()
			running = false
			for _, k in ipairs(AUTO_KEYS) do cfg[k] = false end
			cfg.AutoCapture = false
			restoreInstant()
		end)
	end,
})

-- Detecção do jogo atual (carrega só os scripts que combinam)
task.spawn(function()
	for _, def in ipairs(GameModules) do
		if matchesGame(def, nil) then loadGame(def) end
	end
	local ok, info = pcall(function() return MarketplaceService:GetProductInfo(game.PlaceId) end)
	local gameName = ok and info and info.Name or nil
	GameNameVal.Text = gameName or "Unknown"
	for _, def in ipairs(GameModules) do
		if matchesGame(def, gameName) then loadGame(def) end
	end
end)

-- ================================================================= --
-- 12. LOOPS PRINCIPAIS
-- ================================================================= --
track(UserInputService.JumpRequest:Connect(function()
	if State.InfJump then
		local hum = getHum()
		if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
	end
end))

-- Noclip: lista em cache (não varre GetDescendants todo frame)
track(RunService.Stepped:Connect(function()
	if not State.Noclip then return end
	local now = os.clock()
	if now - noclipRefresh > 1.5 then
		noclipRefresh = now
		rebuildNoclip()
	end
	for i = 1, #noclipParts do
		local p = noclipParts[i]
		if p.CanCollide then
			noclipTouched[p] = true
			p.CanCollide = false
		end
	end
end))

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

	-- Fly
	if State.Fly then
		if flyObjs and flyObjs.bv and flyObjs.bv.Parent then
			local cam = workspace.CurrentCamera
			local root = getRoot()
			if cam and root then
				local mv = getMoveVector()
				local dir = cam.CFrame.RightVector * mv.X - cam.CFrame.LookVector * mv.Z
				if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.yAxis end
				if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) or UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
					dir = dir - Vector3.yAxis
				end
				if dir.Magnitude > 1 then dir = dir.Unit end
				flyVelocity = flyVelocity:Lerp(dir * State.FlySpeed, 1 - math.exp(-12 * dt))
				flyObjs.bv.Velocity = flyVelocity
				local flat = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
				if flat.Magnitude > 0.01 then
					flyObjs.bg.CFrame = CFrame.lookAt(root.Position, root.Position + flat)
				end
			end
		elseif not flyObjs then
			startFly()
		end
	end

	-- Lighting (só escreve quando o valor mudou)
	if State.Fullbright and fbOrig then
		if Lighting.Brightness ~= 2 then Lighting.Brightness = 2 end
		if Lighting.ClockTime ~= 14 then Lighting.ClockTime = 14 end
		if Lighting.GlobalShadows then Lighting.GlobalShadows = false end
		local amb = Color3.fromRGB(178, 178, 178)
		if Lighting.Ambient ~= amb then Lighting.Ambient = amb end
		if Lighting.OutdoorAmbient ~= amb then Lighting.OutdoorAmbient = amb end
	end
	if State.NoFog and fogOrig then
		if Lighting.FogStart ~= 1e6 then Lighting.FogStart = 1e6 end
		if Lighting.FogEnd ~= 1e6 then Lighting.FogEnd = 1e6 end
		for atm in pairs(fogOrig.Atmos) do
			if atm.Parent and atm.Density ~= 0 then atm.Density = 0 end
		end
	end
end))

local frameCount, lastStat = 0, os.clock()
local snowAcc = 0
track(RunService.RenderStepped:Connect(function(dt)
	if State.FovOn then
		local cam = workspace.CurrentCamera
		if cam and cam.FieldOfView ~= State.Fov then cam.FieldOfView = State.Fov end
	end

	-- Neve a ~24 FPS, só com menu aberto
	if isOpen and State.Snow then
		snowAcc = snowAcc + dt
		if snowAcc >= 1 / 24 then
			updateSnow(snowAcc)
			snowAcc = 0
		end
	else
		snowAcc = 0
	end

	frameCount = frameCount + 1
	local now = os.clock()
	if now - lastStat >= 1 then
		if isOpen then
			FpsVal.Text = tostring(frameCount)
			local elapsed = os.time() - startTime
			if elapsed >= 3600 then
				SessionVal.Text = string.format("%dh %02dm", math.floor(elapsed / 3600), math.floor((elapsed % 3600) / 60))
			else
				SessionVal.Text = string.format("%02dm %02ds", math.floor(elapsed / 60), elapsed % 60)
			end
			local count = #Players:GetPlayers() .. "/" .. Players.MaxPlayers
			PlayersVal.Text = count
			SrvPlayersVal.Text = count
			local age = math.floor(workspace.DistributedGameTime)
			local ageText = string.format("%dh %02dm %02ds", math.floor(age / 3600), math.floor((age % 3600) / 60), age % 60)
			AgeVal.Text = ageText
			SrvAgeVal.Text = ageText
			local ok, ping = pcall(function() return Stats.Network.ServerStatsItem["Data Ping"]:GetValue() end)
			if not ok or not ping then ping = LocalPlayer:GetNetworkPing() * 1000 end
			PingVal.Text = math.floor(ping) .. "ms"
		end
		frameCount = 0
		lastStat = now
	end
end))

track(LocalPlayer.CharacterAdded:Connect(function(char)
	cHum, cRoot = nil, nil
	Original.Speed, Original.Jump = nil, nil
	noclipTouched = {}
	table.clear(noclipParts)
	noclipRefresh = 0
	flyObjs = nil
	flyVelocity = Vector3.zero
	if State.Fly then
		task.spawn(function()
			char:WaitForChild("Humanoid", 8)
			char:WaitForChild("HumanoidRootPart", 8)
			task.wait(0.4)
			if State.Fly and LocalPlayer.Character == char then startFly() end
		end)
	end
end))

-- ================================================================= --
-- 13. ABRIR/FECHAR, ARRASTE, UNLOAD E INICIALIZAÇÃO
-- ================================================================= --
local animToken = 0

local function setOpen(state)
	if state == isOpen then return end
	isOpen = state
	animToken = animToken + 1
	local token = animToken

	if isOpen then
		currentScale = computeScale()
		hubCenter = clampCenter(hubCenter)
		MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
		MainFrame.Visible = true
		SnowContainer.Visible = State.Snow
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

-- Keybind
track(UserInputService.InputBegan:Connect(function(input, gp)
	if listeningKey and input.UserInputType == Enum.UserInputType.Keyboard then
		listeningKey = false
		if input.KeyCode ~= Enum.KeyCode.Escape then toggleKey = input.KeyCode end
		keyBtnRef.Text = toggleKey.Name
		tween(keyBtnRef, 0.15, {BackgroundColor3 = COLORS.CardHeader})
		Notify("Keybind", "Menu key: " .. toggleKey.Name, 2)
		return
	end
	if not gp and input.KeyCode == toggleKey then
		setOpen(not isOpen)
	end
end))

-- Arraste
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

-- Hub (arrasta um "fantasma" e só move a janela ao soltar = sem lag)
local hubMoved, hubStartCenter, ghostCenter = false, hubCenter, hubCenter
local function attachHubDrag(handle)
	makeDrag(handle,
		function() hubMoved = false; hubStartCenter = hubCenter; ghostCenter = hubCenter end,
		function(delta)
			if not hubMoved and delta.Magnitude > 3 then
				hubMoved = true
				InputBlocker.Visible = true
				DragGhost.Size = UDim2.fromOffset(hubPixelSize().X, hubPixelSize().Y)
				DragGhost.Visible = true
			end
			if hubMoved then
				ghostCenter = clampCenter(hubStartCenter + delta)
				DragGhost.Position = UDim2.fromOffset(ghostCenter.X, ghostCenter.Y)
			end
		end,
		function()
			if hubMoved then
				hubMoved = false
				hubCenter = ghostCenter
				DragGhost.Visible = false
				InputBlocker.Visible = false
				tween(MainFrame, 0.3, {Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)})
			end
		end
	)
end
attachHubDrag(Sidebar)
attachHubDrag(TopHeader)

-- Botão flutuante (arrasta e clica)
local fbMoved, fbStart = false, Vector2.zero
makeDrag(FloatingBtn,
	function()
		fbMoved = false
		fbStart = Vector2.new(FloatingBtn.AbsolutePosition.X, FloatingBtn.AbsolutePosition.Y)
	end,
	function(delta)
		if not fbMoved and delta.Magnitude > 5 then
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

-- Unload
UnloadHub = function()
	for _, fn in ipairs(cleanupFns) do pcall(fn) end
	State.Fly = false
	stopFly()
	restoreSpeed()
	restoreJump()
	restoreNoclip()
	restoreFullbright()
	restoreFog()
	restoreLowGfx()
	pcall(function() RunService:Set3dRenderingEnabled(true) end)
	if Original.Fov and workspace.CurrentCamera then workspace.CurrentCamera.FieldOfView = Original.Fov end
	pcall(function() LocalPlayer.CameraMaxZoomDistance = origMaxZoom end)
	if afkConn then afkConn:Disconnect(); afkConn = nil end
	if rejoinConn then rejoinConn:Disconnect(); rejoinConn = nil end
	for _, c in ipairs(connections) do pcall(function() c:Disconnect() end) end
	connections = {}
	env.__ShadowHubUnload = nil
	if ScreenGui then ScreenGui:Destroy() end
end
env.__ShadowHubUnload = UnloadHub

-- Inicialização
currentScale = computeScale()
do
	local s = screenSize()
	hubCenter = clampCenter(Vector2.new(s.X / 2, s.Y / 2))
end
MainFrame.Position = UDim2.fromOffset(hubCenter.X, hubCenter.Y)
HubScale.Scale = currentScale

setActiveTab("Home")

task.delay(0.3, function()
	setOpen(true)
	Notify("Dark Shadow Hub", "Loaded. Press " .. toggleKey.Name .. " to toggle.", 3)
end)
