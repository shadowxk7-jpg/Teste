-- ================================================================= --
--  UNIVERSAL: AIMBOT PARA JOGOS DE TIRO  (aba "Aimbot")
--  Funciona em qualquer jogo de tiro (inclusive os que você criar):
--  só usa Players, Humanoid, câmera e raycast padrão do Roblox.
--
--  REGRA: nada roda até você ligar o botão "Aimbot". Ao desligar, tudo
--  é desconectado e todos os efeitos somem da tela.
--
--  COMO ENCAIXAR: cole este bloco INTEIRO no seu script, logo ACIMA de:
--      -- 12. JOGO: MERGE A MINI ARMY
--  (depois do bloco "11. SISTEMA DE SCRIPTS POR JOGO"). Não mexe em
--  nenhuma outra linha do seu script.
-- ================================================================= --
do
	local LOCK_COLOR = Color3.fromRGB(255, 72, 104)
	local PART_MODES = {"Head", "Torso", "Closest", "Root"}
	local KEY_MODES  = {"Hold: Right Mouse", "Hold: Left Mouse", "Always On"}
	local METHODS    = {"Camera", "Mouse (mousemoverel)"}

	local Cfg = {
		FOV = 150, Smooth = 35, MaxDist = 1500, Predict = 0, Deadzone = 2,
		TeamCheck = true, WallCheck = true, Sticky = true, NPCs = false, Trigger = false,
		ShowFov = true, ShowTracer = true, ShowReticle = true, Highlight = true,
		KillFx = true, LockSound = true, Rainbow = false, PauseMenu = true, EspEnemies = false,
		PartMode = 1, KeyMode = 1, Method = 1,
	}

	local Aim = {
		on = false, closing = false, intro = 0, holding = false, touchHeld = false,
		target = nil, part = nil, name = "", dist = 0, lockedTarget = nil, lastScreen = nil,
		locks = 0, kills = 0, fx = nil, conns = {}, hl = nil,
		npcT = 0, npcList = {}, lastTrig = 0, hue = 0, hudText = "", warnedMouse = false, lastErr = 0,
	}

	local rayParams = RaycastParams.new()
	rayParams.FilterType = Enum.RaycastFilterType.Exclude
	rayParams.IgnoreWater = true

	local VIM
	pcall(function() VIM = game:GetService("VirtualInputManager") end)

	-- ======================================================
	-- 1) AUXILIARES
	-- ======================================================
	local function cam() return workspace.CurrentCamera end

	local function ease(t) return 1 - (1 - t) ^ 3 end

	-- Centro da mira: centro da tela (shift lock / 1ª pessoa / celular) ou posição do mouse
	local function aimCenter()
		local c = cam()
		if not c then return Vector2.new() end
		if IS_TOUCH or UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter then
			return c.ViewportSize / 2
		end
		return UserInputService:GetMouseLocation()
	end

	local function isEnemy(p)
		if p == LocalPlayer then return false end
		if Cfg.TeamCheck and not LocalPlayer.Neutral and not p.Neutral then
			if LocalPlayer.Team ~= nil then
				if p.Team == LocalPlayer.Team then return false end
			elseif p.TeamColor == LocalPlayer.TeamColor then
				return false
			end
		end
		return true
	end

	local function alive(char)
		local h = char:FindFirstChildOfClass("Humanoid")
		return h == nil or h.Health > 0
	end

	local function refreshNPCs()
		local now = os.clock()
		if now - Aim.npcT < 1.5 then return Aim.npcList end
		Aim.npcT = now
		local list = {}
		local function consider(m)
			if m:IsA("Model") and m ~= LocalPlayer.Character and not Players:GetPlayerFromCharacter(m)
				and m:FindFirstChildOfClass("Humanoid") then
				list[#list + 1] = m
			end
		end
		for _, c in ipairs(workspace:GetChildren()) do
			consider(c)
			if c:IsA("Folder") or (c:IsA("Model") and not c:FindFirstChildOfClass("Humanoid")) then
				for _, g in ipairs(c:GetChildren()) do
					consider(g)
					if #list > 80 then break end
				end
			end
			if #list > 80 then break end
		end
		Aim.npcList = list
		return list
	end

	local function eachEnemy(fn)
		for _, p in ipairs(Players:GetPlayers()) do
			if isEnemy(p) and p.Character then fn(p.Character, p.DisplayName) end
		end
		if Cfg.NPCs then
			for _, m in ipairs(refreshNPCs()) do
				if m.Parent then fn(m, m.Name) end
			end
		end
	end

	local function candidateParts(char, root)
		local head = char:FindFirstChild("Head")
		local torso = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso") or root
		local mode = PART_MODES[Cfg.PartMode]
		if mode == "Head" then return {head or root} end
		if mode == "Torso" then return {torso} end
		if mode == "Root" then return {root} end
		local list = {}
		if head then list[#list + 1] = head end
		if torso then list[#list + 1] = torso end
		if root and root ~= torso then list[#list + 1] = root end
		return list
	end

	-- Parede no caminho? (partes quase transparentes não bloqueiam)
	local function visible(part, char)
		if not Cfg.WallCheck then return true end
		local c = cam()
		local origin = c.CFrame.Position
		local mine = LocalPlayer.Character
		rayParams.FilterDescendantsInstances = mine and {mine, c} or {c}
		local res = workspace:Raycast(origin, part.Position - origin, rayParams)
		if not res then return true end
		local hit = res.Instance
		if hit:IsDescendantOf(char) then return true end
		return hit.Transparency >= 0.9
	end

	local function evaluate(char, center, fov)
		local root = char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
		if not root or not alive(char) then return nil end
		local c = cam()
		local d3 = (root.Position - c.CFrame.Position).Magnitude
		if d3 > Cfg.MaxDist then return nil end
		local bestPart, bestScr
		for _, part in ipairs(candidateParts(char, root)) do
			local v, onScreen = c:WorldToViewportPoint(part.Position)
			if onScreen and v.Z > 0 then
				local d = (Vector2.new(v.X, v.Y) - center).Magnitude
				if d <= fov and (not bestScr or d < bestScr) then bestPart, bestScr = part, d end
			end
		end
		if not bestPart or not visible(bestPart, char) then return nil end
		return bestPart, bestScr, d3
	end

	local function fireShot(center)
		if mouse1click then
			pcall(mouse1click)
		elseif VIM then
			pcall(function()
				VIM:SendMouseButtonEvent(center.X, center.Y, 0, true, game, 0)
				VIM:SendMouseButtonEvent(center.X, center.Y, 0, false, game, 0)
			end)
		end
	end

	-- ======================================================
	-- 2) EFEITOS VISUAIS (criados só ao ligar, destruídos ao desligar)
	-- ======================================================
	local function buildFx()
		local fx = {tweens = {}, tintFrames = {}, tintStrokes = {}}
		local function loop(inst, time, props, style, rev)
			local t = TweenService:Create(inst, TweenInfo.new(time, style or Enum.EasingStyle.Linear,
				Enum.EasingDirection.InOut, -1, rev or false), props)
			t:Play()
			fx.tweens[#fx.tweens + 1] = t
		end

		fx.root = create("Frame", {
			Name = "AimbotFX", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
			ZIndex = 30, Active = false, Parent = ScreenGui,
		})

		-- Anel de FOV: brilho externo + anel principal com "cometa" girando
		fx.glow = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), BackgroundTransparency = 1, ZIndex = 30, Parent = fx.root,
		})
		corner(fx.glow, 1000)
		fx.glowStroke = stroke(fx.glow, COLORS.Cyan, 7, 0.88)

		fx.fov = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), BackgroundColor3 = COLORS.Accent,
			BackgroundTransparency = 0.94, BorderSizePixel = 0, ZIndex = 31, Parent = fx.root,
		})
		corner(fx.fov, 1000)
		fx.fovStroke = stroke(fx.fov, COLORS.Cyan, 2, 0)
		fx.fovGrad = create("UIGradient", {
			Transparency = NumberSequence.new({
				NumberSequenceKeypoint.new(0, 0.85), NumberSequenceKeypoint.new(0.18, 0),
				NumberSequenceKeypoint.new(0.5, 0.85), NumberSequenceKeypoint.new(1, 0.85),
			}),
			Parent = fx.fovStroke,
		})
		loop(fx.fovGrad, 2.2, {Rotation = 360})

		-- marcas N/E/S/O no anel
		for _, def in ipairs({{0.5, 0, 2, 9}, {1, 0.5, 9, 2}, {0.5, 1, 2, 9}, {0, 0.5, 9, 2}}) do
			local tick = create("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(def[1], def[2]),
				Size = UDim2.fromOffset(def[3], def[4]), BackgroundColor3 = COLORS.Cyan,
				BorderSizePixel = 0, ZIndex = 32, Parent = fx.fov,
			})
			fx.tintFrames[#fx.tintFrames + 1] = tick
		end

		-- ponto central
		fx.dot = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(4, 4),
			BackgroundColor3 = COLORS.Cyan, BorderSizePixel = 0, ZIndex = 32, Parent = fx.root,
		})
		corner(fx.dot, 2)
		fx.tintFrames[#fx.tintFrames + 1] = fx.dot

		-- Linha (tracer) do centro até o alvo
		fx.tracer = create("Frame", {
			AnchorPoint = Vector2.new(0, 0.5), Size = UDim2.fromOffset(0, 1), BackgroundColor3 = COLORS.Cyan,
			BorderSizePixel = 0, Visible = false, ZIndex = 31, Parent = fx.root,
		})
		create("UIGradient", {
			Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.9), NumberSequenceKeypoint.new(1, 0)}),
			Parent = fx.tracer,
		})

		-- Retículo no alvo: anel + marcas giratórias + nome + vida
		fx.ret = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(54, 54), BackgroundTransparency = 1,
			Visible = false, ZIndex = 32, Parent = fx.root,
		})
		corner(fx.ret, 100)
		fx.retStroke = stroke(fx.ret, COLORS.Cyan, 2, 0.05)
		fx.tintStrokes[#fx.tintStrokes + 1] = fx.retStroke
		fx.retSpin = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
			Size = UDim2.fromScale(1.4, 1.4), BackgroundTransparency = 1, ZIndex = 32, Parent = fx.ret,
		})
		for _, def in ipairs({{0.5, 0, 3, 11}, {1, 0.5, 11, 3}, {0.5, 1, 3, 11}, {0, 0.5, 11, 3}}) do
			local tick = create("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(def[1], def[2]),
				Size = UDim2.fromOffset(def[3], def[4]), BackgroundColor3 = COLORS.Cyan,
				BorderSizePixel = 0, ZIndex = 33, Parent = fx.retSpin,
			})
			fx.tintFrames[#fx.tintFrames + 1] = tick
		end
		loop(fx.retSpin, 3, {Rotation = 360})
		fx.retName = label({
			AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 1, 16), Size = UDim2.fromOffset(180, 14),
			Text = "", Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Center,
			TextStrokeTransparency = 0.35, ZIndex = 33, Parent = fx.ret,
		})
		fx.retBarBg = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 1, 33), Size = UDim2.fromOffset(72, 4),
			BackgroundColor3 = Color3.fromRGB(8, 14, 36), BorderSizePixel = 0, ZIndex = 33, Parent = fx.ret,
		})
		corner(fx.retBarBg, 2)
		fx.retBar = create("Frame", {
			Size = UDim2.fromScale(1, 1), BackgroundColor3 = COLORS.Green, BorderSizePixel = 0,
			ZIndex = 34, Parent = fx.retBarBg,
		})
		corner(fx.retBar, 2)

		-- Pílula de status no topo (desliza ao ligar/desligar)
		fx.hud = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, -44), Size = UDim2.fromOffset(240, 30),
			BackgroundColor3 = COLORS.Sidebar, BackgroundTransparency = 0.08, BorderSizePixel = 0,
			ZIndex = 33, Parent = fx.root,
		})
		corner(fx.hud, 15)
		fx.hudStroke = stroke(fx.hud, COLORS.Cyan, 1.5, 0.15)
		fx.tintStrokes[#fx.tintStrokes + 1] = fx.hudStroke
		fx.hudDot = create("Frame", {
			Size = UDim2.fromOffset(8, 8), Position = UDim2.new(0, 14, 0.5, -4), BackgroundColor3 = COLORS.Cyan,
			BorderSizePixel = 0, ZIndex = 34, Parent = fx.hud,
		})
		corner(fx.hudDot, 4)
		fx.tintFrames[#fx.tintFrames + 1] = fx.hudDot
		loop(fx.hudDot, 0.7, {BackgroundTransparency = 0.75}, Enum.EasingStyle.Sine, true)
		fx.hudLabel = label({
			Size = UDim2.new(1, -40, 1, 0), Position = UDim2.new(0, 30, 0, 0), Text = "AIMBOT  ·  SCANNING",
			Font = Enum.Font.GothamBold, TextSize = 11, TextTruncate = Enum.TextTruncate.AtEnd,
			ZIndex = 34, Parent = fx.hud,
		})

		-- Botão AIM para celular (segure para mirar)
		if IS_TOUCH then
			fx.touchBtn = create("TextButton", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.86, 0, 0.58, 0), Size = UDim2.fromOffset(64, 64),
				BackgroundColor3 = COLORS.CardHeader, BackgroundTransparency = 0.15, Text = "AIM",
				TextColor3 = COLORS.TextMain, Font = Enum.Font.GothamBlack, TextSize = 14,
				AutoButtonColor = false, ZIndex = 40, Parent = fx.root,
			})
			corner(fx.touchBtn, 32)
			fx.touchStroke = stroke(fx.touchBtn, COLORS.Cyan, 2, 0.1)
			fx.touchBtn.InputBegan:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
					Aim.touchHeld = true
					fx.touchStroke.Color = LOCK_COLOR
				end
			end)
			fx.touchBtn.InputEnded:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
					Aim.touchHeld = false
					fx.touchStroke.Color = COLORS.Cyan
				end
			end)
		end

		return fx
	end

	local function pulseAt(fx, pos, color)
		local r = create("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(pos.X, pos.Y), Size = UDim2.fromOffset(24, 24),
			BackgroundTransparency = 1, ZIndex = 34, Parent = fx.root,
		})
		corner(r, 100)
		local s = stroke(r, color, 2.5, 0)
		tween(r, 0.5, {Size = UDim2.fromOffset(130, 130)}, Enum.EasingStyle.Quad)
		tween(s, 0.5, {Transparency = 1})
		task.delay(0.55, function() r:Destroy() end)
	end

	local function killBurst(fx, pos)
		for i = 1, 14 do
			local a = math.rad(i * (360 / 14) + math.random(-12, 12))
			local d = math.random(40, 95)
			local sz = math.random(3, 6)
			local p = create("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(pos.X, pos.Y), Size = UDim2.fromOffset(sz, sz),
				BackgroundColor3 = (i % 2 == 0) and COLORS.Cyan or LOCK_COLOR, BorderSizePixel = 0, ZIndex = 35, Parent = fx.root,
			})
			corner(p, 3)
			tween(p, 0.6, {
				Position = UDim2.fromOffset(pos.X + math.cos(a) * d, pos.Y + math.sin(a) * d),
				BackgroundTransparency = 1, Size = UDim2.fromOffset(1, 1),
			}, Enum.EasingStyle.Quad)
			task.delay(0.65, function() p:Destroy() end)
		end
		local t = label({
			AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(pos.X, pos.Y - 30), Size = UDim2.fromOffset(180, 18),
			Text = "✦ ELIMINATED ✦", Font = Enum.Font.GothamBlack, TextSize = 13, TextColor3 = LOCK_COLOR,
			TextXAlignment = Enum.TextXAlignment.Center, TextStrokeTransparency = 0.4, ZIndex = 36, Parent = fx.root,
		})
		tween(t, 0.9, {Position = UDim2.fromOffset(pos.X, pos.Y - 70), TextTransparency = 1, TextStrokeTransparency = 1}, Enum.EasingStyle.Quad)
		task.delay(0.95, function() t:Destroy() end)
	end

	-- ======================================================
	-- 3) DESLIGAR / LIMPAR TUDO
	-- ======================================================
	local function syncEsp()
		ESP.SetEnabled("AimEnemies", Aim.on and Cfg.EspEnemies)
	end

	local function hardStop()
		Aim.on, Aim.closing = false, false
		pcall(function() RunService:UnbindFromRenderStep("ShadowAimbot") end)
		for _, cn in ipairs(Aim.conns) do pcall(function() cn:Disconnect() end) end
		Aim.conns = {}
		Aim.holding, Aim.touchHeld = false, false
		if Aim.hl then pcall(function() Aim.hl:Destroy() end); Aim.hl = nil end
		local fx = Aim.fx
		Aim.fx = nil
		if fx then
			for _, t in ipairs(fx.tweens) do pcall(function() t:Cancel() end) end
			pcall(function() fx.root:Destroy() end)
		end
		Aim.target, Aim.part, Aim.lockedTarget, Aim.lastScreen = nil, nil, nil, nil
		ESP.SetEnabled("AimEnemies", false)
	end

	-- ======================================================
	-- 4) LOOP PRINCIPAL (só existe enquanto o Aimbot está ligado)
	-- ======================================================
	local function step(dt)
		local fx = Aim.fx
		if not fx then return end
		local c = cam()

		if Aim.on then Aim.intro = math.min(1, Aim.intro + dt * 3)
		else Aim.intro = Aim.intro - dt * 3.5 end
		local e = ease(math.clamp(Aim.intro, 0, 1))
		Aim.hue = (Aim.hue + dt * 0.25) % 1

		local paused = Cfg.PauseMenu and isOpen
		fx.root.Visible = not paused
		local center = aimCenter()

		-- anel de FOV (cresce ao ligar, encolhe ao desligar)
		local diameter = Cfg.FOV * 2 * e
		fx.fov.Size = UDim2.fromOffset(diameter, diameter)
		fx.fov.Position = UDim2.fromOffset(center.X, center.Y)
		fx.glow.Size = UDim2.fromOffset(diameter + 12, diameter + 12)
		fx.glow.Position = fx.fov.Position
		fx.fov.Visible = Cfg.ShowFov
		fx.glow.Visible = Cfg.ShowFov
		fx.dot.Position = fx.fov.Position
		fx.dot.Visible = e > 0.3
		fx.hud.Position = UDim2.new(0.5, 0, 0, -44 + 54 * e)
		if fx.touchBtn then fx.touchBtn.Visible = e > 0.2 end

		-- desligando: só anima a saída e limpa
		if not Aim.on then
			fx.ret.Visible = false
			fx.tracer.Visible = false
			if Aim.hl then Aim.hl.Adornee = nil end
			if Aim.intro <= 0 then hardStop() end
			return
		end
		if not c then return end

		-- validade do alvo / detecção de kill
		local tgt = Aim.target
		if tgt then
			if not tgt.Parent then
				Aim.target = nil
			else
				local h = tgt:FindFirstChildOfClass("Humanoid")
				if h and h.Health <= 0 then
					if Aim.lockedTarget == tgt then
						Aim.kills = Aim.kills + 1
						if Cfg.KillFx and Aim.lastScreen then killBurst(fx, Aim.lastScreen) end
						if Cfg.LockSound then Sfx.Play("Notify") end
					end
					Aim.target = nil
				end
			end
		end

		local myHum = getHum()
		local meAlive = myHum == nil or myHum.Health > 0
		local held = (not paused) and (Cfg.KeyMode == 3 or Aim.holding or Aim.touchHeld)

		-- escolha do alvo (sticky: mantém o atual enquanto ainda estiver perto da mira)
		local part, scr, d3
		if meAlive and not paused then
			if Cfg.Sticky and Aim.target then
				part, scr, d3 = evaluate(Aim.target, center, Cfg.FOV * 1.3)
			end
			if not part then
				local bestChar, bestPart, bestScr, bestName, bestD
				eachEnemy(function(char, name)
					local p2, s2, dd = evaluate(char, center, Cfg.FOV)
					if p2 and (not bestScr or s2 < bestScr) then
						bestChar, bestPart, bestScr, bestName, bestD = char, p2, s2, name, dd
					end
				end)
				if bestChar then
					if Aim.target ~= bestChar then Aim.name = bestName end
					Aim.target, part, scr, d3 = bestChar, bestPart, bestScr, bestD
				else
					Aim.target = nil
				end
			end
		else
			Aim.target = nil
		end
		Aim.part, Aim.dist = part, d3 or 0

		-- estado: idle / preview / locked
		local state = (not Aim.target) and "idle" or (held and "locked" or "preview")
		local col = (state == "locked") and LOCK_COLOR or (Cfg.Rainbow and Color3.fromHSV(Aim.hue, 0.65, 1) or COLORS.Cyan)
		fx.fovStroke.Color = col
		fx.glowStroke.Color = col
		fx.retStroke.Color = col
		fx.hudStroke.Color = col
		for _, f in ipairs(fx.tintFrames) do f.BackgroundColor3 = col end

		-- efeito ao travar em um novo alvo
		if held and Aim.target then
			if Aim.lockedTarget ~= Aim.target then
				Aim.lockedTarget = Aim.target
				Aim.locks = Aim.locks + 1
				if Cfg.LockSound then Sfx.Play("On") end
				if Aim.lastScreen then pulseAt(fx, Aim.lastScreen, LOCK_COLOR) end
				fx.ret.Size = UDim2.fromOffset(92, 92)
				tween(fx.ret, 0.28, {Size = UDim2.fromOffset(54, 54)}, Enum.EasingStyle.Back)
			end
		else
			Aim.lockedTarget = nil
		end

		-- texto da pílula
		local hudText
		if state == "locked" then hudText = "AIMBOT  ·  LOCKED  ·  " .. Aim.name
		elseif state == "preview" then hudText = "AIMBOT  ·  TARGET  ·  " .. Aim.name
		else hudText = "AIMBOT  ·  SCANNING" end
		if hudText ~= Aim.hudText then
			Aim.hudText = hudText
			fx.hudLabel.Text = hudText
		end

		-- sem alvo: esconde retículo, tracer e highlight
		if not (Aim.target and part) then
			fx.ret.Visible = false
			fx.tracer.Visible = false
			if Aim.hl then Aim.hl.Adornee = nil end
			return
		end

		-- posição do alvo na tela (com predição opcional)
		local aimPos = part.Position
		if Cfg.Predict > 0 then
			aimPos = aimPos + part.AssemblyLinearVelocity * ((c.CFrame.Position - aimPos).Magnitude / Cfg.Predict)
		end
		local v = c:WorldToViewportPoint(aimPos)
		local sp = Vector2.new(v.X, v.Y)
		Aim.lastScreen = sp

		-- retículo + nome + vida
		fx.ret.Visible = Cfg.ShowReticle
		fx.ret.Position = UDim2.fromOffset(sp.X, sp.Y)
		fx.retName.Text = Aim.name .. "  ·  " .. math.floor(d3 or 0) .. " st"
		fx.retName.TextColor3 = col
		local h = Aim.target:FindFirstChildOfClass("Humanoid")
		local frac = h and math.clamp(h.Health / math.max(h.MaxHealth, 1), 0, 1) or 1
		fx.retBar.Size = UDim2.fromScale(frac, 1)
		fx.retBar.BackgroundColor3 = Color3.fromHSV(0.33 * frac, 0.8, 1)

		-- tracer
		local delta = sp - center
		local len = delta.Magnitude
		fx.tracer.Visible = Cfg.ShowTracer and len > 4
		fx.tracer.Position = UDim2.fromOffset(center.X, center.Y)
		fx.tracer.Size = UDim2.fromOffset(len, state == "locked" and 2 or 1)
		fx.tracer.Rotation = math.deg(math.atan2(delta.Y, delta.X))
		fx.tracer.BackgroundColor3 = col

		-- highlight no corpo do alvo
		if Cfg.Highlight then
			if not Aim.hl then
				Aim.hl = create("Highlight", {
					FillTransparency = 0.72, OutlineTransparency = 0.1,
					DepthMode = Enum.HighlightDepthMode.AlwaysOnTop, Parent = ESPFolder,
				})
			end
			if Aim.hl.Adornee ~= Aim.target then Aim.hl.Adornee = Aim.target end
			Aim.hl.FillColor = col
			Aim.hl.OutlineColor = col
		elseif Aim.hl then
			Aim.hl.Adornee = nil
		end

		-- mira de verdade (só enquanto estiver segurando)
		if held then
			if len > Cfg.Deadzone then
				local alpha = 1 - (math.clamp(Cfg.Smooth, 0, 95) / 100) ^ (dt * 60)
				if Cfg.Method == 2 and mousemoverel then
					local mv = delta * alpha
					pcall(mousemoverel, mv.X, mv.Y)
				else
					if Cfg.Method == 2 and not mousemoverel and not Aim.warnedMouse then
						Aim.warnedMouse = true
						Notify("Aimbot", "Executor sem mousemoverel. Usando método Camera.", 4)
					end
					c.CFrame = c.CFrame:Lerp(CFrame.lookAt(c.CFrame.Position, aimPos), alpha)
				end
			end
			if Cfg.Trigger and len <= math.max(8, Cfg.Deadzone + 6) and os.clock() - Aim.lastTrig > 0.09 then
				Aim.lastTrig = os.clock()
				fireShot(center)
			end
		end
	end

	local function safeStep(dt)
		local ok, err = pcall(step, dt)
		if not ok and os.clock() - Aim.lastErr > 5 then
			Aim.lastErr = os.clock()
			warn("[ShadowHub] Aimbot: " .. tostring(err))
		end
	end

	-- ======================================================
	-- 5) LIGAR / DESLIGAR
	-- ======================================================
	local function start()
		if Aim.on then return end
		if Aim.closing or Aim.fx then hardStop() end
		Aim.on, Aim.closing, Aim.intro = true, false, 0
		Aim.hudText, Aim.lockedTarget, Aim.target = "", nil, nil
		Aim.fx = buildFx()
		table.insert(Aim.conns, UserInputService.InputBegan:Connect(function(input)
			if Cfg.KeyMode == 1 and input.UserInputType == Enum.UserInputType.MouseButton2 then
				Aim.holding = true
			elseif Cfg.KeyMode == 2 and input.UserInputType == Enum.UserInputType.MouseButton1 then
				Aim.holding = true
			end
		end))
		table.insert(Aim.conns, UserInputService.InputEnded:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton2 or input.UserInputType == Enum.UserInputType.MouseButton1 then
				Aim.holding = false
			end
		end))
		pcall(function() RunService:UnbindFromRenderStep("ShadowAimbot") end)
		RunService:BindToRenderStep("ShadowAimbot", Enum.RenderPriority.Camera.Value + 1, safeStep)
		syncEsp()
	end

	local function stop()
		if not Aim.on then return end
		Aim.on, Aim.closing = false, true
		for _, cn in ipairs(Aim.conns) do pcall(function() cn:Disconnect() end) end
		Aim.conns = {}
		Aim.holding, Aim.touchHeld = false, false
		syncEsp()
		task.delay(1.5, function()
			if Aim.closing then hardStop() end
		end)
	end

	-- ESP de inimigos (só aparece com o Aimbot ligado)
	ESP.Register("AimEnemies", function()
		local out = {}
		if not (Aim.on and Cfg.EspEnemies) then return out end
		eachEnemy(function(char, name)
			local root = char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
			if root and alive(char) then
				out[#out + 1] = {part = root, hl = char, text = name,
					color = (char == Aim.target) and LOCK_COLOR or COLORS.Cyan}
			end
		end)
		return out
	end)

	table.insert(cleanupFns, function() hardStop() end)

	-- ======================================================
	-- 6) INTERFACE: aba "Aimbot"
	-- ======================================================
	local page = createTab("Aimbot", ICONS.Combat, false, 35)
	local group = createSubGroup("Aimbot", page)
	local pMain = group.Add("Main", ICONS.Bolt)
	local pTarget = group.Add("Target", ICONS.Search)
	local pVisual = group.Add("Visuals", ICONS.Shield)
	local pExtra = group.Add("Extra", ICONS.Settings)

	local labels = {}

	-- MAIN
	local secPower = AddSection(pMain, "Power", "Aimbot", "Main")
	AddToggle(secPower, "Aimbot", "Off = nothing runs and nothing shows on screen", false, function(on)
		if on then
			start()
			Notify("Aimbot", "Active. " .. (IS_TOUCH and "Hold the AIM button." or KEY_MODES[Cfg.KeyMode]), 3, true)
		else
			stop()
		end
	end)

	local secInfo = AddSection(pMain, "Status", "Aimbot", "Main")
	labels.status = AddInfo(secInfo, "State", "Off")
	labels.target = AddInfo(secInfo, "Target", "-")
	labels.dist = AddInfo(secInfo, "Distance", "-")
	labels.locks = AddInfo(secInfo, "Locks", "0")
	labels.kills = AddInfo(secInfo, "Eliminations", "0")

	local secInput = AddSection(pMain, "Input", "Aimbot", "Main")
	local btnKey, btnMethod
	btnKey = AddButton(secInput, "Aim key: " .. KEY_MODES[Cfg.KeyMode], function()
		Cfg.KeyMode = Cfg.KeyMode % #KEY_MODES + 1
		Aim.holding = false
		btnKey.Text = "Aim key: " .. KEY_MODES[Cfg.KeyMode]
	end)
	btnMethod = AddButton(secInput, "Method: " .. METHODS[Cfg.Method], function()
		Cfg.Method = Cfg.Method % #METHODS + 1
		btnMethod.Text = "Method: " .. METHODS[Cfg.Method]
	end)
	AddText(secInput, "Camera = gira a câmera (funciona na maioria dos jogos). Mouse = move o mouse (use se o jogo trava a câmera). No celular, segure o botão AIM que aparece na tela.")

	-- TARGET
	local secAim = AddSection(pTarget, "Aim", "Aimbot", "Target")
	AddSlider(secAim, "FOV radius", 30, 600, Cfg.FOV, 5, " px", function(v) Cfg.FOV = v end)
	AddSlider(secAim, "Smoothness", 0, 95, Cfg.Smooth, 1, "%", function(v) Cfg.Smooth = v end)
	AddSlider(secAim, "Max distance", 100, 5000, Cfg.MaxDist, 50, " st", function(v) Cfg.MaxDist = v end)
	AddSlider(secAim, "Prediction speed (0 = off)", 0, 1500, Cfg.Predict, 25, " st/s", function(v) Cfg.Predict = v end)
	AddSlider(secAim, "Deadzone", 0, 30, Cfg.Deadzone, 1, " px", function(v) Cfg.Deadzone = v end)
	local btnPart
	btnPart = AddButton(secAim, "Aim part: " .. PART_MODES[Cfg.PartMode], function()
		Cfg.PartMode = Cfg.PartMode % #PART_MODES + 1
		btnPart.Text = "Aim part: " .. PART_MODES[Cfg.PartMode]
	end)

	local secRules = AddSection(pTarget, "Rules", "Aimbot", "Target")
	AddToggle(secRules, "Team check", "Ignores teammates", Cfg.TeamCheck, function(on) Cfg.TeamCheck = on end, true)
	AddToggle(secRules, "Wall check", "Only targets you can see", Cfg.WallCheck, function(on) Cfg.WallCheck = on end, true)
	AddToggle(secRules, "Sticky target", "Keeps the target until it leaves the FOV", Cfg.Sticky, function(on) Cfg.Sticky = on end, true)
	AddToggle(secRules, "Include NPCs / bots", "Humanoids that are not players", Cfg.NPCs, function(on)
		Cfg.NPCs = on
		Aim.npcT = 0
	end, true)

	-- VISUALS
	local secFx = AddSection(pVisual, "Effects", "Aimbot", "Visuals")
	AddToggle(secFx, "FOV ring", "Glowing ring with a rotating comet", Cfg.ShowFov, function(on) Cfg.ShowFov = on end, true)
	AddToggle(secFx, "Target reticle", "Rotating reticle, name, distance and health", Cfg.ShowReticle, function(on) Cfg.ShowReticle = on end, true)
	AddToggle(secFx, "Tracer line", "Line from your aim to the target", Cfg.ShowTracer, function(on) Cfg.ShowTracer = on end, true)
	AddToggle(secFx, "Target highlight", "Glow around the target's body", Cfg.Highlight, function(on) Cfg.Highlight = on end, true)
	AddToggle(secFx, "Kill effect", "Burst + ELIMINATED text", Cfg.KillFx, function(on) Cfg.KillFx = on end, true)
	AddToggle(secFx, "Lock sound", "Sound when locking and on kills", Cfg.LockSound, function(on) Cfg.LockSound = on end, true)
	AddToggle(secFx, "Rainbow ring", "Cycles the ring colors", Cfg.Rainbow, function(on) Cfg.Rainbow = on end, true)

	local secEsp = AddSection(pVisual, "Enemies", "Aimbot", "Visuals")
	AddToggle(secEsp, "Enemy ESP", "Highlights every enemy while Aimbot is on", Cfg.EspEnemies, function(on)
		Cfg.EspEnemies = on
		syncEsp()
	end, true)

	-- EXTRA
	local secExtra = AddSection(pExtra, "Extra", "Aimbot", "Extra")
	AddToggle(secExtra, "Pause when hub is open", "Aim and effects hide while this menu is open", Cfg.PauseMenu, function(on) Cfg.PauseMenu = on end, true)
	AddToggle(secExtra, "Triggerbot", "Clicks automatically when the target is on your crosshair", Cfg.Trigger, function(on) Cfg.Trigger = on end, true)
	AddText(secExtra, "O Triggerbot só dispara enquanto você estiver mirando (segurando a tecla de mira) e o alvo estiver bem no centro. Usa mouse1click ou clique virtual.")

	-- Atualiza os textos de status (só com a aba aberta)
	task.spawn(function()
		while ScreenGui.Parent do
			task.wait(0.4)
			if isOpen and tabs.Aimbot and tabs.Aimbot.Active then
				local held = Aim.on and (Cfg.KeyMode == 3 or Aim.holding or Aim.touchHeld)
				labels.status.Text = not Aim.on and "Off" or (Aim.target and (held and "Locked" or "Target in FOV") or "Scanning")
				labels.target.Text = (Aim.on and Aim.target) and Aim.name or "-"
				labels.dist.Text = (Aim.on and Aim.target) and (math.floor(Aim.dist) .. " st") or "-"
				labels.locks.Text = tostring(Aim.locks)
				labels.kills.Text = tostring(Aim.kills)
			end
		end
	end)
end
