-- ================================================================= --
-- 12. JOGO: MERGE A MINI ARMY   (v2: Auto Merge corrigido)
--  Sub-abas: Quick | Merge | Economy | Combat | ESP | Tools
--
--  O QUE FOI CORRIGIDO NO AUTO MERGE
--   1) Antes, unidades sem Humanoid/atributos eram ignoradas (nenhum par
--      era encontrado). Agora qualquer modelo repetido na sua base conta.
--   2) Se a base não for detectada, o script usa a SUA posição como base
--      (antes ele ficava parado esperando "Set Base Here").
--   3) Descoberta automática do remote: o script testa os remotes com nome
--      de merge no jogo e confere se a unidade sumiu/mudou.
--   4) Aprendizado passivo: se você mesclar 1 vez à mão, ele copia o comando.
--   5) Botão "Merge Diagnosis" copia um relatório para depuração.
-- ================================================================= --
RegisterGame({
	Name = "Merge a Mini Army",
	TabName = "Mini Army",
	Icon = ICONS.Merge,
	PlaceIds = {},
	GameIds = {},
	Names = {"Merge a Mini Army", "Merge Mini Army", "Mini Army"},
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
		local RS = game:GetService("ReplicatedStorage")
		local running = true
		local stats = {clicks = 0, rebirths = 0, merges = 0, fails = 0, captures = 0}
		local ctl = {}
		local MERGE_WORDS = {"merge", "combine", "fuse"}

		local cfg = {
			Merge = false, MergeDelay = 0.1, Batch = 3, UseRemote = true, AutoLearn = true,
			AutoDiscover = true, AutoBaseHere = true,
			Walk = false,
			Drag = false, MergeBtn = false, UnitWords = {}, BaseRadius = 70,
			Upgrade = false, UpgradeEvery = 1, UpgradeWords = {}, Rebirth = false, RebirthEvery = 20,
			Collect = false, Spin = false, Equip = false, Buy = false, ActionDelay = 0.8,
			Silent = true,
			MouseFallback = false,
			Deploy = false, DeployEvery = 5, Battle = false, Attack = false, AtkEvent = true, AtkCamp = true,
			AtkBase = false, AtkTeleport = true, KeepDeployed = true, AtkTimeout = 12,
			AutoCapture = false, Range = 25, Instant = false,
			Custom = false, CustomWords = {}, AfkRebirth = false,
		}

		local function isPlayerChar(i) return i:IsA("Model") and Players:GetPlayerFromCharacter(i) ~= nil end
		local function posOf(i)
			if i:IsA("BasePart") then return i.Position end
			if i:IsA("Model") then return i:GetPivot().Position end
			return nil
		end

		-- ======================================================
		-- A) MINHA BASE
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

		-- CORREÇÃO: se não achar a base, usa a posição atual em vez de travar
		local function ensureBase(quiet)
			if base.center then return true end
			if detectBase() then return true end
			if cfg.AutoBaseHere and setBaseHere() then
				if not quiet then ctx.Notify("Base", "Not detected. Using your position as base", 4) end
				return true
			end
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
		local BAD_NAMES = {"tree", "wall", "fence", "floor", "tile", "slot", "plot", "spawn", "pad", "grass",
			"decor", "rock", "bush", "gate", "road", "baseplate", "button", "door", "sign", "lamp", "bridge", "terrain"}
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
					if modelSize(cur) < 24 then best = cur else break end
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

		-- Chave de "unidades iguais". Retorna (chave, strict).
		-- strict = certeza de que é unidade; senão só conta se houver 2+ iguais.
		local function unitKey(m)
			local ln = string.lower(m.Name)
			if #cfg.UnitWords == 0 and hasAny(ln, BAD_NAMES) then return nil end
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
			local name = (string.gsub(m.Name, "%x%x%x%x%x%x%x%x%-%x%x%x%x.*$", ""))
			name = (string.gsub(name, "%d%d%d%d+", "")) -- ids longos; níveis (1-3 dígitos) ficam
			if meta then
				name = (string.gsub(name, "%d+", ""))
			elseif not string.find(name, "%d") then
				for _, d in ipairs(m:GetDescendants()) do
					if d:IsA("TextLabel") and string.find(d.Text, "%d") then
						parts[#parts + 1] = "t:" .. d.Text
						break
					end
				end
			end
			local strict = meta
				or (m:FindFirstChildOfClass("Humanoid") ~= nil)
				or (m:FindFirstChildOfClass("AnimationController") ~= nil)
			if #cfg.UnitWords > 0 then
				strict = hasAny(ln, cfg.UnitWords)
				if not strict then return nil end
			end
			table.sort(parts)
			return name .. "|" .. table.concat(parts, ","), strict
		end

		local function scanUnits(force)
			local now = os.clock()
			if not force and now - scan.t < (Perf.Lite and 2 or 0.6) then return scan end
			scan.t = now
			local groups, strictKey, seen = {}, {}, {}
			local function consider(m)
				if seen[m] or not m.Parent then return end
				seen[m] = true
				if isPlayerChar(m) then return end
				if not inMyBase(m) then return end
				local key, strict = unitKey(m)
				if key then
					local g = groups[key]
					if not g then g = {}; groups[key] = g end
					g[#g + 1] = m
					if strict then strictKey[key] = true end
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
			local total, pairsN = 0, 0
			for key, g in pairs(groups) do
				if #g < 2 and not strictKey[key] then
					groups[key] = nil
				else
					total = total + #g
					pairsN = pairsN + math.floor(#g / 2)
				end
			end
			scan.groups, scan.units, scan.pairs = groups, total, pairsN
			return scan
		end

		local function pickPairs(maxN)
			local sc = scanUnits()
			local now = os.clock()
			local out = {}
			for _, list in pairs(sc.groups) do
				local alive = {}
				for _, u in ipairs(list) do
					if u.Parent and (not cooldown[u] or cooldown[u] < now) then alive[#alive + 1] = u end
				end
				while #alive >= 2 and #out < maxN do
					local a = table.remove(alive, 1)
					local pa = unitPart(a)
					local bi, bd
					for i = 1, #alive do
						local pb = unitPart(alive[i])
						if pa and pb then
							local d = (pa.Position - pb.Position).Magnitude
							if not bd or d < bd then bi, bd = i, d end
						elseif not bi then
							bi = i
						end
					end
					local b = table.remove(alive, bi or 1)
					out[#out + 1] = {a, b}
				end
				if #out >= maxN then break end
			end
			return out
		end

		-- ======================================================
		-- C) EXECUÇÃO DO MERGE (sem teleporte)
		-- ======================================================
		local learned = env.__ShadowMiniLearn
		local recording, captured = false, nil

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

		local function touchPair(a, b)
			local pa, pb = unitPart(a), unitPart(b)
			if not (pa and pb) then return end
			if cfg.Drag then pcall(function() a:PivotTo(CFrame.new(pb.Position)) end) end
			if firetouchinterest then
				local root = getRoot()
				pcall(function()
					if root then
						firetouchinterest(root, pa, 0); firetouchinterest(root, pa, 1)
						firetouchinterest(root, pb, 0); firetouchinterest(root, pb, 1)
					end
					firetouchinterest(pa, pb, 0); firetouchinterest(pa, pb, 1)
				end)
			end
			firePrompts(a)
			firePrompts(b)
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

		local function teleportMerge(a, b)
			local root = getRoot()
			if not root then return end
			local back = root.CFrame
			for _, u in ipairs({a, b}) do
				local p = unitPart(u)
				if p and p.Parent then
					root.CFrame = CFrame.new(p.Position + Vector3.new(0, 1.5, 0))
					root.AssemblyLinearVelocity = Vector3.zero
					if firetouchinterest then
						pcall(function() firetouchinterest(root, p, 0); firetouchinterest(root, p, 1) end)
					end
					firePrompts(u)
					task.wait(0.06)
				end
			end
			root.CFrame = back
			root.AssemblyLinearVelocity = Vector3.zero
		end

		local function fireOne(a, b)
			local used = false
			if cfg.UseRemote and learned and not learned.direct and #learned.inst >= 1 then
				used = fireLearned(a, b)
			end
			-- Sempre também tenta o toque: se o remote não fizer nada, o toque faz
			touchPair(a, b)
		end

		-- ---------- Descoberta automática do remote ----------
		local remoteCands = {t = 0, list = {}}
		local function remoteCandidates()
			if os.clock() - remoteCands.t < 20 then return remoteCands.list end
			remoteCands.t = os.clock()
			local list, i = {}, 0
			for _, d in ipairs(RS:GetDescendants()) do
				if (d:IsA("RemoteEvent") or d:IsA("RemoteFunction")) and hasAny(string.lower(d.Name), MERGE_WORDS) then
					list[#list + 1] = d
				end
				i = i + 1
				if i % 800 == 0 then task.wait() end
			end
			remoteCands.list = list
			return list
		end

		local discovering, lastDiscover = false, 0
		local function discoverRemote(a, b)
			if discovering or learned then return end
			discovering = true
			lastDiscover = os.clock()
			local cands = remoteCandidates()
			local keyA = unitKey(a)
			local function changed()
				return (not a.Parent) or (not b.Parent) or (unitKey(a) ~= keyA)
			end
			local shapes = {{1, 2}, {2, 1}, "direct"}
			for _, r in ipairs(cands) do
				for _, shape in ipairs(shapes) do
					if learned or not running then break end
					if a.Parent and b.Parent then
						local args
						if shape == "direct" then args = table.pack() else args = table.pack(a, b) end
						local method = r:IsA("RemoteFunction") and "InvokeServer" or "FireServer"
						task.spawn(function()
							pcall(function()
								if method == "InvokeServer" then r:InvokeServer(table.unpack(args, 1, args.n))
								else r:FireServer(table.unpack(args, 1, args.n)) end
							end)
						end)
						task.wait(0.45)
						if changed() then
							learned = {remote = r, method = method, args = args,
								inst = (shape == "direct") and {} or shape, byName = true, direct = (shape == "direct")}
							env.__ShadowMiniLearn = learned
							if Refs.mRemote then Refs.mRemote.Text = r.Name end
							ctx.Notify("Remote found", r.Name, 4)
						end
					end
				end
				if learned then break end
			end
			discovering = false
		end

		local function runBatch(list)
			local keys = {}
			for i, p in ipairs(list) do
				keys[i] = unitKey(p[1])
				fireOne(p[1], p[2])
			end
			task.wait(0.3)
			local now = os.clock()
			local retry
			for i, p in ipairs(list) do
				local a, b = p[1], p[2]
				local merged = (not a.Parent) or (not b.Parent) or (unitKey(a) ~= keys[i])
				if merged then
					stats.merges = stats.merges + 1
					failCount[a], failCount[b] = nil, nil
				else
					local n = (failCount[a] or 0) + 1
					failCount[a], failCount[b] = n, n
					local t = now + math.min(1.5 * n, 10)
					cooldown[a], cooldown[b] = t, t
					stats.fails = stats.fails + 1
					if n >= 3 and cfg.Walk and not retry then retry = p end
					-- 2 falhas seguidas e nenhum remote conhecido: tenta descobrir
					if n == 2 and cfg.AutoDiscover and not learned and not discovering and os.clock() - lastDiscover > 15 then
						task.spawn(discoverRemote, a, b)
					end
				end
			end
			if retry then teleportMerge(retry[1], retry[2]) end
			scan.t = 0
		end

		local mergeOnce = false
		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				if cfg.Merge or mergeOnce then
					if ensureBase(true) then
						if cfg.UseRemote and learned and learned.direct and learned.remote then
							local sc = scanUnits()
							if sc.pairs > 0 then
								fireLearned()
								stats.merges = stats.merges + 1
							else
								mergeOnce = false
							end
							task.wait(math.max(cfg.MergeDelay, Perf.Lite and 0.4 or 0.1))
						else
							local list = pickPairs(cfg.Batch)
							if #list > 0 then
								local ok, err = pcall(runBatch, list)
								if not ok then warn("[ShadowHub] merge: " .. tostring(err)) end
								task.wait(math.max(cfg.MergeDelay, Perf.Lite and 0.3 or 0.03))
							else
								mergeOnce = false
								task.wait(0.35)
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

		-- ---------- Aprendizado do remote (passivo + manual) ----------
		local SKIP_REMOTE = {"position", "move", "camera", "ping", "heartbeat", "analytics", "log", "cursor", "mouse", "look"}
		local function onRemote(remote, method, ...)
			if not recording or learned then return end
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
			if byName or #inst >= 2 then
				if not captured or byName then
					captured = {remote = remote, method = method, args = args, inst = inst,
						byName = byName, direct = (#inst == 0)}
					-- grava na hora (não espera o loop de tempo)
					learned = captured
					env.__ShadowMiniLearn = learned
					recording = false
					task.defer(function()
						if Refs.mRemote then Refs.mRemote.Text = learned.remote.Name end
						ctx.Notify("Learned", learned.remote.Name .. (learned.direct and " (direct)" or " (targets)"), 4)
					end)
				end
			end
		end

		local function startLearning(seconds, quiet)
			if recording then return end
			if not installHook() then
				if not quiet then ctx.Notify("Learn", "Executor lacks hookmetamethod. Touch mode still works", 5) end
				return
			end
			captured, recording = nil, true
			env.__ShadowHookFn = onRemote
			if not quiet then ctx.Notify("Learning", "Merge two units BY HAND now (" .. seconds .. "s)", 6) end
			task.spawn(function()
				local t0 = os.clock()
				while running and recording and os.clock() - t0 < seconds do task.wait(0.25) end
				if recording then
					recording = false
					if not quiet then ctx.Notify("Learn", "Nothing captured. Try again", 4) end
				end
			end)
		end

		-- Aprendizado passivo já ao carregar: um merge manual seu ensina o script
		task.defer(function()
			if not learned and installHook() then
				env.__ShadowHookFn = onRemote
				recording = true
			end
		end)

		-- ======================================================
		-- D) BOTÕES DA INTERFACE DO JOGO
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
		local CONFIRM = {"confirm", "yes"}
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
						if key == "Upgrade" and #cfg.UpgradeWords > 0 and not hasAny(info.text, cfg.UpgradeWords) then key = nil end
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

		local function clickNow(key)
			ensureButtonTracking()
			local waited = 0
			while not scanDone and waited < 3 do task.wait(0.1); waited = waited + 0.1 end
			local n = clickCategory({[key] = true}, os.clock(), key == "Rebirth")
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
		-- F) ESP DO JOGO
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
		-- G) INTERFACE
		-- ======================================================
		local function simple(sec, key, title, desc)
			ctl[key] = AddToggle(sec, title, desc, false, function(on)
				cfg[key] = on
				if on then ensureButtonTracking() end
			end)
		end

		local pq = ctx.Page("Quick", ctx.Icons.Bolt)
		do
			local qs = pq.Section("Main Automation")
			ctl.Merge = AddToggle(qs, "Auto Merge", "No teleport. Merges equal units in your base", false, function(on)
				cfg.Merge = on
				if on then
					ensureBase()
					if cfg.AutoLearn and not learned and not recording then startLearning(600, true) end
				end
			end)
			ctl.Upgrade = AddToggle(qs, "Auto Upgrade", "Silent. No menu opens on your screen", false, function(on)
				cfg.Upgrade = on
				if on then ensureButtonTracking() end
			end)
			ctl.Rebirth = AddToggle(qs, "Auto Rebirth", "Silent. Careful: resets your progress", false, function(on)
				cfg.Rebirth = on
				if on then ensureButtonTracking() end
			end)
			simple(qs, "Collect", "Auto Collect / Claim", "Silent claim of rewards")
			simple(qs, "Buy", "Auto Buy / Spawn Units", "Silent buy and spawn")

			local qi = pq.Section("Live Status")
			Refs.qBase   = AddInfo(qi, "My base", "Not set")
			Refs.qPairs  = AddInfo(qi, "Pairs ready", "0")
			Refs.qMerges = AddInfo(qi, "Merges done", "0")
			Refs.qClicks = AddInfo(qi, "Silent clicks", "0")
			AddText(qi, "Ligue e ande livremente: o merge acontece sozinho, sem teleporte. Se o merge não pegar de primeira, o script testa os remotes do jogo sozinho. Você também pode fazer 1 merge à mão que ele copia o comando.")
		end

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
					if cfg.AutoLearn and not learned and not recording then startLearning(600, true) end
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
			AddSlider(am, "Pairs per cycle", 1, 6, cfg.Batch, 1, "", function(v) cfg.Batch = v end)
			AddButton(am, "Merge Now", function()
				if ensureBase() then mergeOnce = true else ctx.Notify("Merge", "Set your base first", 3) end
			end)
			Refs.mUnits  = AddInfo(am, "Units in my base", "0")
			Refs.mPairs  = AddInfo(am, "Pairs found", "0")
			Refs.mMerges = AddInfo(am, "Merges done", "0")
			Refs.mFails  = AddInfo(am, "Failed attempts", "0")

			local mb = pm.Section("My Base (merge only here)")
			Refs.mBase = AddInfo(mb, "Base", "Not set")
			AddToggle(mb, "Use my position if base not found", "Avoids the merge waiting forever", true, function(on) cfg.AutoBaseHere = on end)
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
			AddText(mb, "Fique no centro da sua base e toque em Set Base Here se a detecção automática errar. Só unidades dentro dessa área são mescladas.")

			local mm = pm.Section("Method")
			AddToggle(mm, "Use learned remote", "Fastest: replays the real merge call", true, function(on) cfg.UseRemote = on end)
			AddToggle(mm, "Auto-learn remote", "Learns it when you merge by hand once", true, function(on) cfg.AutoLearn = on end)
			AddToggle(mm, "Auto-discover remote", "Tests the game's merge remotes by itself", true, function(on) cfg.AutoDiscover = on end)
			Refs.mRemote = AddInfo(mm, "Learned remote", learned and learned.remote and learned.remote.Name or "None")
			AddButton(mm, "Learn Merge Remote", function() startLearning(20, false) end)
			AddButton(mm, "Discover Remote Now", function()
				if not ensureBase() then return end
				local list = pickPairs(1)
				if #list == 0 then ctx.Notify("Discover", "No pair of equal units found", 4) return end
				ctx.Notify("Discover", "Testing remotes...", 3)
				task.spawn(discoverRemote, list[1][1], list[1][2])
			end)
			AddButton(mm, "Forget learned remote", function()
				learned = nil
				env.__ShadowMiniLearn = nil
				Refs.mRemote.Text = "None"
				if installHook() then env.__ShadowHookFn = onRemote; recording = true end
			end)
			AddToggle(mm, "Teleport fallback", "Only if a pair fails 3x. Goes there and returns", false, function(on) cfg.Walk = on end)
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
			AddText(rb, "Ligue Auto Upgrade e Auto Rebirth na aba Quick. Depois do clique em Rebirth o script também confirma o diálogo (Confirm / Yes).")

			local sl = pe.Section("Silent Mode")
			AddToggle(sl, "Silent clicks", "Works with the game menu closed", true, function(on) cfg.Silent = on end)
			AddToggle(sl, "Mouse fallback", "Virtual mouse if no signal works (visible buttons only)", false, function(on) cfg.MouseFallback = on end)
			AddText(sl, "No modo silencioso o script aciona os botões por trás da interface: nada abre na tela e seu mouse/dedo continua livre.")

			local ex = pe.Section("Extras")
			simple(ex, "Equip", "Auto Equip Best", "Clicks equip best / equip all")
			simple(ex, "Spin", "Auto Spin", "Free spins only (Robux ones are skipped)")
			AddSlider(ex, "Action delay", 0.2, 3, cfg.ActionDelay, 0.1, "s", function(v) cfg.ActionDelay = v end)
			AddText(ex, "Botões ligados a Robux, gamepass, VIP ou presentes são sempre ignorados.")
		end

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
			AddButton(dev, "Merge Diagnosis (copy report)", function()
				ensureBase(true)
				local sc = scanUnits(true)
				local out = {}
				out[#out + 1] = "base: " .. tostring(base.label) .. " r=" .. tostring(math.floor(base.radius))
				out[#out + 1] = "units: " .. sc.units .. " | pairs: " .. sc.pairs
				out[#out + 1] = "merges: " .. stats.merges .. " | fails: " .. stats.fails
				out[#out + 1] = "learned: " .. (learned and learned.remote and learned.remote:GetFullName() or "none")
				out[#out + 1] = "hook: " .. tostring(hookmetamethod ~= nil) .. " touch: " .. tostring(firetouchinterest ~= nil)
					.. " prompt: " .. tostring(fireproximityprompt ~= nil)
				for _, r in ipairs(remoteCandidates()) do out[#out + 1] = "remote: " .. r:GetFullName() end
				for key, list in pairs(sc.groups) do
					out[#out + 1] = #list .. "x | " .. key .. " | " .. list[1]:GetFullName()
				end
				dump(out, "Merge Diagnosis")
			end)
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
				for _, d in ipairs(RS:GetDescendants()) do
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
			AddText(dev, "Se o merge ainda não funcionar: toque em Merge Diagnosis, cole o resultado aqui no chat e eu ajusto para o seu jogo.")
		end

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
			env.__ShadowHookFn = nil
			for _, k in ipairs(DUE_KEYS) do cfg[k] = false end
			cfg.Merge, cfg.Attack, cfg.AutoCapture = false, false, false
			restoreInstant()
		end)
	end,
})

-- ================================================================= --
-- Utilitário compartilhado: INTANGIBILIDADE (portada do script velho)
-- CanTouch/CanQuery desligados nas peças do seu personagem, mantidos
-- a cada frame e restaurados ao desligar, ao morrer e ao descarregar.
-- ================================================================= --
function Mv.newIntangible()
	local self = {on = false, parts = {}, touched = setmetatable({}, {__mode = "k"}), refresh = 0}
	local function rebuild()
		table.clear(self.parts)
		local c = LocalPlayer.Character
		if not c then return end
		for _, p in ipairs(c:GetDescendants()) do
			if p:IsA("BasePart") then self.parts[#self.parts + 1] = p end
		end
	end
	function self.Step()
		if not self.on then return end
		local now = os.clock()
		if now - self.refresh > 1 then self.refresh = now; rebuild() end
		for i = 1, #self.parts do
			local p = self.parts[i]
			if p.Parent then
				if p.CanTouch or p.CanQuery then
					self.touched[p] = true
					p.CanTouch = false
					p.CanQuery = false
				end
			end
		end
	end
	function self.Restore()
		for p in pairs(self.touched) do
			if p.Parent then p.CanTouch = true; p.CanQuery = true end
		end
		self.touched = setmetatable({}, {__mode = "k"})
		table.clear(self.parts)
	end
	function self.Set(on)
		self.on = on and true or false
		if self.on then self.refresh = 0 else self.Restore() end
	end
	return self
end

-- ================================================================= --
-- 12b. JOGO: MURDER MYSTERY 2 (MM2)
--  Detecção (independe de idioma): PlaceId/GameId > nome > conteúdo
--  (o remote "GetPlayerData" e a estrutura do mapa existem em qualquer idioma).
--  Sub-abas: Quick | ESP | Tools
-- ================================================================= --
RegisterGame({
	Name = "Murder Mystery 2",
	TabName = "MM2",
	Icon = ICONS.Combat,
	PlaceIds = {142823291},
	GameIds = {66654135},
	Names = {"Murder Mystery 2", "Murder Mystery", "MM2"},
	Detect = function()
		local RS = game:GetService("ReplicatedStorage")
		if RS:FindFirstChild("GetPlayerData", true) then return true end
		local hasLobby = workspace:FindFirstChild("Lobby") ~= nil
		local hasRemotes = RS:FindFirstChild("Remotes") ~= nil
		local hasGun = workspace:FindFirstChild("GunDrop", true) ~= nil
		return (hasLobby and hasRemotes) or hasGun
	end,
	Build = function(ctx)
		local RS = game:GetService("ReplicatedStorage")
		local running = true
		local cfg = {Coins = false, CoinDelay = 0.35, Gun = false, GunTp = false}
		local intang = Mv.newIntangible()
		local info = {murder = "-", sheriff = "-"}

		-- ---------- Funções (roles) ----------
		local roleMap, roleT = {}, 0
		local function refreshRoles()
			if os.clock() - roleT < 2 then return end
			roleT = os.clock()
			local rem = RS:FindFirstChild("GetPlayerData", true)
			if rem and rem:IsA("RemoteFunction") then
				task.spawn(function()
					local ok, data = pcall(function() return rem:InvokeServer() end)
					if ok and type(data) == "table" then
						local map = {}
						for name, d in pairs(data) do
							if type(d) == "table" and d.Role then map[tostring(name)] = tostring(d.Role) end
						end
						roleMap = map
					end
				end)
			end
		end

		local function toolRole(pl)
			local ch, bp = pl.Character, pl:FindFirstChildOfClass("Backpack")
			local function has(n)
				return (ch and ch:FindFirstChild(n)) or (bp and bp:FindFirstChild(n))
			end
			if has("Knife") or has("Blade") then return "Murderer" end
			if has("Gun") or has("Revolver") then return "Sheriff" end
			return nil
		end

		local function roleOf(pl)
			local r = roleMap[pl.Name]
			if r and r ~= "" and r ~= "nil" then
				if r == "Hero" then return "Sheriff" end
				return r
			end
			return toolRole(pl) or "Innocent"
		end

		local ROLE_COLOR = {
			Murderer = Color3.fromRGB(255, 50, 80),
			Sheriff = Color3.fromRGB(70, 150, 255),
			Innocent = Color3.fromRGB(120, 235, 150),
		}

		ESP.Register("MM2Roles", function()
			refreshRoles()
			local out = {}
			for _, pl in ipairs(Players:GetPlayers()) do
				if pl ~= LocalPlayer then
					local ch = pl.Character
					local root = ch and ch:FindFirstChild("HumanoidRootPart")
					local hum = ch and ch:FindFirstChildOfClass("Humanoid")
					if root and hum and hum.Health > 0 then
						local role = roleOf(pl)
						local tag = (role == "Murderer" and "ASSASSINO") or (role == "Sheriff" and "XERIFE") or nil
						out[#out + 1] = {part = root, hl = ch, color = ROLE_COLOR[role] or ROLE_COLOR.Innocent,
							text = tag and (pl.DisplayName .. " [" .. tag .. "]") or pl.DisplayName}
					end
				end
			end
			return out
		end)

		local gunCache = {t = 0, part = nil}
		local function findGun()
			if os.clock() - gunCache.t < 1 then return gunCache.part end
			gunCache.t = os.clock()
			local g = workspace:FindFirstChild("GunDrop", true)
			if g and not g:IsA("BasePart") then g = g:FindFirstChildWhichIsA("BasePart", true) end
			gunCache.part = g
			return g
		end

		ESP.Register("MM2Gun", function()
			local g = findGun()
			if g and g.Parent then
				return {{part = g, hl = nil, text = "ARMA DROPADA", color = COLORS.Gold}}
			end
			return {}
		end)

		-- ---------- Coletas ----------
		local coinBox = {t = 0, folder = nil}
		local function findCoins()
			if os.clock() - coinBox.t > 4 or not (coinBox.folder and coinBox.folder.Parent) then
				coinBox.t = os.clock()
				coinBox.folder = workspace:FindFirstChild("CoinContainer", true)
			end
			return coinBox.folder
		end

		local function grabGun()
			local root = getRoot()
			local g = findGun()
			if not (root and g and g.Parent) then return false end
			if firetouchinterest then
				pcall(function() firetouchinterest(root, g, 0); firetouchinterest(root, g, 1) end)
			end
			if cfg.GunTp then
				local back = root.CFrame
				root.CFrame = CFrame.new(g.Position + Vector3.new(0, 2, 0))
				task.wait(0.12)
				root.CFrame = back
			end
			return true
		end

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				if cfg.Gun then pcall(grabGun) end
				task.wait(cfg.Gun and 0.25 or 0.5)
			end
		end)

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				if cfg.Coins and firetouchinterest then
					pcall(function()
						local root = getRoot()
						local box = findCoins()
						if root and box then
							for _, c in ipairs(box:GetChildren()) do
								if not (cfg.Coins and running) then break end
								if c:IsA("BasePart") and c.Parent then
									firetouchinterest(root, c, 0)
									firetouchinterest(root, c, 1)
									task.wait(cfg.CoinDelay)
								end
							end
						end
					end)
				end
				task.wait(0.5)
			end
		end)

		ctx.track(RunService.Stepped:Connect(function() intang.Step() end))
		ctx.track(LocalPlayer.CharacterAdded:Connect(function()
			intang.Restore()
			if intang.on then intang.refresh = 0 end
		end))

		-- ---------- Interface ----------
		local pq = ctx.Page("Quick", ctx.Icons.Bolt)
		do
			local sec = pq.Section("Round")
			Refs.mmMurder = AddInfo(sec, "Murderer", "-")
			Refs.mmSheriff = AddInfo(sec, "Sheriff", "-")
			AddText(sec, "Os nomes aparecem quando a rodada começa e o ESP de funções está ligado.")

			local au = pq.Section("Automation")
			AddToggle(au, "Auto Grab Gun", "Picks up the dropped gun without moving", false, function(on)
				cfg.Gun = on
			end)
			AddToggle(au, "Gun: teleport fallback", "Goes to the gun and returns instantly", false, function(on) cfg.GunTp = on end)
			AddToggle(au, "Auto Collect Coins", "Touches coins in the map", false, function(on)
				cfg.Coins = on
				if on and not firetouchinterest then ctx.Notify("Coins", "Executor lacks firetouchinterest", 4) end
			end)
			AddSlider(au, "Coin delay", 0.1, 1.5, cfg.CoinDelay, 0.05, "s", function(v) cfg.CoinDelay = v end)

			local ig = pq.Section("Intangibility")
			AddToggle(ig, "Obito Intangibility", "Attacks and touches pass through you (CanTouch/CanQuery off)", false, function(on)
				intang.Set(on)
			end)
			AddText(ig, "Mantém seu personagem intangível a toques e consultas. Desliga sozinho ao descarregar o hub.")
		end

		local px = ctx.Page("ESP", ctx.Icons.Search)
		do
			local es = px.Section("Role ESP")
			AddToggle(es, "Roles ESP (Murderer / Sheriff)", "Red = murderer, blue = sheriff, green = innocent", false, function(on)
				ESP.SetEnabled("MM2Roles", on)
			end)
			AddToggle(es, "Dropped Gun ESP", "Marks the sheriff's dropped gun", false, function(on)
				ESP.SetEnabled("MM2Gun", on)
			end)
			AddSlider(es, "ESP max distance", 100, 2000, ESP.MaxDist, 50, "", function(v) ESP.MaxDist = v end)
		end

		local pt = ctx.Page("Tools", ctx.Icons.Shield)
		do
			local ds = pt.Section("Developer Tools")
			AddButton(ds, "Copy Roles (debug)", function()
				refreshRoles()
				task.wait(0.5)
				local out = {}
				for _, pl in ipairs(Players:GetPlayers()) do
					out[#out + 1] = pl.Name .. " = " .. roleOf(pl)
				end
				local text = table.concat(out, "\n")
				if ctx.copy then ctx.copy(text); ctx.Notify("Roles", "Copied", 2) else print(text) end
			end)
		end

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(1)
				local m, s = "-", "-"
				for _, pl in ipairs(Players:GetPlayers()) do
					local r = roleOf(pl)
					if r == "Murderer" then m = pl.DisplayName elseif r == "Sheriff" then s = pl.DisplayName end
				end
				info.murder, info.sheriff = m, s
				if ctx.IsVisible() then
					Refs.mmMurder.Text = m
					Refs.mmSheriff.Text = s
				end
			end
		end)

		ctx.OnUnload(function()
			running = false
			cfg.Coins, cfg.Gun = false, false
			intang.Set(false)
		end)
	end,
})

-- ================================================================= --
-- 12c. JOGO: NINJA TYCOON
--  Detecção: PlaceId/nome, ou pelo CONTEÚDO (objetos "Essentials/Giver"
--  e prompts de Scroll), que não dependem do idioma da interface.
--  Dica: abra Home > Copy Game Info e coloque o PlaceId abaixo para
--  reconhecimento instantâneo.
--  Sub-abas: Quick | ESP
-- ================================================================= --
RegisterGame({
	Name = "Ninja Tycoon",
	TabName = "Ninja Tycoon",
	Icon = ICONS.Economy,
	PlaceIds = {},          -- opcional: coloque aqui o PlaceId do Ninja Tycoon
	GameIds = {},
	Names = {"Ninja Tycoon", "Ninja Tycon", "Ninja Tycoon!"},
	Detect = function()
		local i = 0
		local giver, scroll = false, false
		for _, d in ipairs(workspace:GetDescendants()) do
			if d.Name == "Giver" and d.Parent and d.Parent.Name == "Essentials" then giver = true end
			if d:IsA("ProximityPrompt") then
				local t = string.lower(d.ObjectText .. " " .. d.ActionText .. " " .. (d.Parent and d.Parent.Name or ""))
				if string.find(t, "scroll", 1, true) then scroll = true end
			end
			if giver then return true end
			i = i + 1
			if i % 1500 == 0 then task.wait() end
		end
		return giver or scroll
	end,
	Build = function(ctx)
		local running = true
		local cfg = {Money = false, MoneyEvery = 1, Scrolls = false, ScrollTp = true, Instant = false}
		local intang = Mv.newIntangible()
		local counts = {givers = 0, scrolls = 0, collected = 0}

		local givers = {t = 0, list = {}}
		local function refreshGivers()
			if os.clock() - givers.t < 5 then return givers.list end
			givers.t = os.clock()
			local list, i = {}, 0
			for _, d in ipairs(workspace:GetDescendants()) do
				if d.Name == "Giver" and d:IsA("BasePart") and d.Parent and d.Parent.Name == "Essentials" then
					list[#list + 1] = d
				end
				i = i + 1
				if i % 1500 == 0 then task.wait() end
			end
			givers.list = list
			counts.givers = #list
			return list
		end

		local scrollsC = {t = 0, list = {}}
		local function refreshScrolls()
			if os.clock() - scrollsC.t < 3 then return scrollsC.list end
			scrollsC.t = os.clock()
			local list, i = {}, 0
			for _, d in ipairs(workspace:GetDescendants()) do
				if d:IsA("ProximityPrompt") then
					local t = string.lower(d.ObjectText .. " " .. d.ActionText .. " " .. (d.Parent and d.Parent.Name or ""))
					if string.find(t, "scroll", 1, true) then list[#list + 1] = d end
				end
				i = i + 1
				if i % 1500 == 0 then task.wait() end
			end
			scrollsC.list = list
			counts.scrolls = #list
			return list
		end

		local function partOfPrompt(p)
			local par = p.Parent
			if par and par:IsA("BasePart") then return par end
			if par and par:IsA("Model") then return par.PrimaryPart or par:FindFirstChildWhichIsA("BasePart", true) end
			if par and par:IsA("Attachment") and par.Parent:IsA("BasePart") then return par.Parent end
			return nil
		end

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				if cfg.Money and firetouchinterest then
					pcall(function()
						local root = getRoot()
						if root then
							for _, g in ipairs(refreshGivers()) do
								if g.Parent then
									firetouchinterest(root, g, 0)
									task.wait()
									firetouchinterest(root, g, 1)
									counts.collected = counts.collected + 1
								end
							end
						end
					end)
				end
				task.wait(cfg.MoneyEvery)
			end
		end)

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				if cfg.Scrolls and fireproximityprompt then
					pcall(function()
						for _, p in ipairs(refreshScrolls()) do
							if not cfg.Scrolls then break end
							local part = partOfPrompt(p)
							local root = getRoot()
							if p.Parent and p.Enabled and part and root then
								if cfg.ScrollTp then
									local back = root.CFrame
									root.CFrame = part.CFrame
									task.wait(0.1)
									fireproximityprompt(p)
									task.wait(0.1)
									root.CFrame = back
								else
									fireproximityprompt(p)
								end
								counts.collected = counts.collected + 1
							end
						end
					end)
				end
				task.wait(0.5)
			end
		end)

		local held = {}
		local function applyInstantAll(on)
			for _, p in ipairs(refreshScrolls()) do
				if p.Parent then
					if on then
						if p.HoldDuration > 0 then held[p] = p.HoldDuration; p.HoldDuration = 0 end
					elseif held[p] then
						p.HoldDuration = held[p]
					end
				end
			end
			if not on then held = {} end
		end

		ESP.Register("NinjaScrolls", function()
			local out = {}
			for _, p in ipairs(refreshScrolls()) do
				local part = p.Parent and partOfPrompt(p)
				if part then
					out[#out + 1] = {part = part, hl = nil, text = "SCROLL", color = COLORS.Gold}
				end
			end
			return out
		end)

		ctx.track(RunService.Stepped:Connect(function() intang.Step() end))
		ctx.track(LocalPlayer.CharacterAdded:Connect(function()
			intang.Restore()
			if intang.on then intang.refresh = 0 end
		end))

		local pq = ctx.Page("Quick", ctx.Icons.Bolt)
		do
			local au = pq.Section("Automation")
			AddToggle(au, "Auto Collect Money", "Touches every Giver (Essentials)", false, function(on)
				cfg.Money = on
				if on and not firetouchinterest then ctx.Notify("Money", "Executor lacks firetouchinterest", 4) end
			end)
			AddSlider(au, "Collect delay", 0.3, 5, cfg.MoneyEvery, 0.1, "s", function(v) cfg.MoneyEvery = v end)
			AddToggle(au, "Auto Collect Scrolls", "Fires Scroll prompts", false, function(on)
				cfg.Scrolls = on
				if on and not fireproximityprompt then ctx.Notify("Scrolls", "Executor lacks fireproximityprompt", 4) end
			end)
			AddToggle(au, "Scrolls: teleport and return", "Goes to the scroll and back instantly", true, function(on) cfg.ScrollTp = on end)
			AddToggle(au, "Instant Scroll Interact", "Removes hold time of scroll prompts", false, function(on)
				cfg.Instant = on
				applyInstantAll(on)
			end)

			local ig = pq.Section("Intangibility / Anti-Skill")
			AddToggle(ig, "Obito Intangibility (Ghost)", "Skills and touches pass through you", false, function(on)
				intang.Set(on)
			end)
			AddText(ig, "Seu personagem fica intangível a toques e consultas (mesma técnica do script velho). Combine com Noclip da aba Player se quiser atravessar paredes.")

			local st = pq.Section("Status")
			Refs.nGivers = AddInfo(st, "Givers found", "0")
			Refs.nScrolls = AddInfo(st, "Scrolls found", "0")
			Refs.nCollected = AddInfo(st, "Collected", "0")
		end

		local px = ctx.Page("ESP", ctx.Icons.Search)
		do
			local es = px.Section("ESP")
			AddToggle(es, "Scrolls ESP", "Marks scrolls in the map", false, function(on)
				ESP.SetEnabled("NinjaScrolls", on)
			end)
			AddToggle(es, "Players ESP", "Highlight + name + distance", false, function(on)
				ESP.SetEnabled("Players", on)
				if Refs.espPlayers then Refs.espPlayers.Set(on, true) end
			end)
			AddSlider(es, "ESP max distance", 100, 2000, ESP.MaxDist, 50, "", function(v) ESP.MaxDist = v end)
		end

		task.spawn(function()
			while running and ctx.ScreenGui.Parent do
				task.wait(1)
				if ctx.IsVisible() then
					Refs.nGivers.Text = tostring(counts.givers)
					Refs.nScrolls.Text = tostring(counts.scrolls)
					Refs.nCollected.Text = tostring(counts.collected)
				end
			end
		end)

		ctx.OnUnload(function()
			running = false
			cfg.Money, cfg.Scrolls = false, false
			intang.Set(false)
			applyInstantAll(false)
		end)
	end,
})
