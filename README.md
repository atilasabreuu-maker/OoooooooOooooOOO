--[[
	PRYPOT 0.X1 — ARMADURA-ESQUELETO ANIMATRÔNICA (veículo estacionado, parkour arcade, escavação)
	LocalScript ÚNICO, 100% MOBILE-FIRST, SEM DEPENDÊNCIAS EXTERNAS
	==============================================================================================
	ONDE COLOCAR: StarterPlayer > StarterPlayerScripts > novo LocalScript (cole tudo).
	Tudo (modelo, animações, efeitos, UI, cutscene) é gerado por código. Nenhum asset, módulo ou servidor.

	RESUMO
	  • A armadura é um VEÍCULO no Workspace: fica parada no mapa (desligada, cabeça baixa) e continua lá
	    quando você sai. Não depende das partes do R15: só usa Humanoid + HumanoidRootPart (funciona
	    em R15 e não quebra em outros rigs).
	  • ENTRAR dispara uma CUTSCENE: a estrutura nas costas da armadura solta agulhas neurais que
	    perfuram as costelas do operador, puxam o corpo para dentro, as costelas se fecham, a luz neural
	    sobe pela coluna até o crânio, os olhos acendem e ela acorda convulsionando.
	  • Parkour arcade: corrida rápida, passos/degraus automáticos (IK nos pés), pulo carregado (superpulo),
	    pulo duplo, DASH aéreo/terrestre, MANTLE (sobe em bordas), agarrar em QUALQUER parede/teto/rampa,
	    correr em paredes, wall-kick, mergulho-impacto (slam) com cratera.
	  • ESCAVAR: perfura o chão e anda por baixo dele (só a coluna com engrenagens aparece), deixando
	    montinhos e crateras que SE REPARAM sozinhos em segundos. Há um SISTEMA DE SEGURANÇA que analisa o
	    solo (vazio, solo fino, limite do mapa, borda) e cria uma "área compatível" (piso de segurança)
	    para o jogador nunca cair no void.
	  • SOCOS que quebram paredes e objetos (apenas visual, só para você) e se reconstroem depois.
	  • MODO FURTIVO: agachada/aranha, quase invisível, silenciosa.
	  • Pés com MOLAS que absorvem impacto; engrenagens, pistões, cabos com física, dedos-lâmina,
	    braços auxiliares que agarram a parede, olhos que seguem, mandíbula, LEDs, vapor.

	CONTROLES
	  Celular : botões na tela (PULO segura=superpulo, SOCO, DASH, GARRA segura, ESCAVAR, FURTIVO, ENTRAR/SAIR)
	  PC      : Espaço pulo (segurar=superpulo) | F soco | Shift dash | Q segurar=agarrar | R escavar
	            C furtivo | E entrar/sair | V pular cutscene
	  Console : A pulo | X soco | B dash | L1 garra | Y escavar | R3 furtivo | DPadUp interagir

	LIMITAÇÃO: por ser só LocalScript, tudo é visível apenas para VOCÊ (inclusive paredes quebradas).
]]

--------------------------------------------------------------------------------
-- SERVIÇOS
--------------------------------------------------------------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local CollectionService = game:GetService("CollectionService")

local player = Players.LocalPlayer

--------------------------------------------------------------------------------
-- CONFIGURAÇÃO (ajuste aqui)
--------------------------------------------------------------------------------
local CFG = {
	Name = "PRYPOT 0.X1",
	Spawn = { Offset = Vector3.new(0, 0, -18), Fixed = nil }, -- Offset relativo ao seu 1º spawn; Fixed = CFrame fixo
	EnterRange = 14, -- distância para o botão ENTRAR aparecer

	Keys = {
		Interact = Enum.KeyCode.E, Dash = Enum.KeyCode.LeftShift, Punch = Enum.KeyCode.F, Grip = Enum.KeyCode.Q,
		Dig = Enum.KeyCode.R, Stealth = Enum.KeyCode.C, Jump = Enum.KeyCode.Space, Skip = Enum.KeyCode.V,
	},
	Pad = {
		Interact = Enum.KeyCode.DPadUp, Dash = Enum.KeyCode.ButtonB, Punch = Enum.KeyCode.ButtonX, Grip = Enum.KeyCode.ButtonL1,
		Dig = Enum.KeyCode.ButtonY, Stealth = Enum.KeyCode.ButtonR3, Jump = Enum.KeyCode.ButtonA, Skip = Enum.KeyCode.ButtonSelect,
	},

	-- G = altura do centro do RootPart até o chão (HipHeight = G - RootSize.Y/2)
	Body = { G = 4.8, CrouchG = 3.5, StealthG = 3.7, RootSize = Vector3.new(2.6, 4.2, 2.0) },
	Move = { Run = 44, Stealth = 16, Air = 1.0 },
	Jump = { Normal = 84, Super = 178, ChargeTime = 0.8, Double = 78, Coyote = 0.12, Buffer = 0.12 },
	Dash = { Speed = 100, Time = 0.2, Cost = 16, Cooldown = 0.55 },
	Grip = { Speed = 42, DrainMove = 6, DrainIdle = 3, Reach = 3.6, Stand = 2.0, KickAway = 60, KickUp = 76, AutoWall = true },
	Mantle = { Min = 1.9, Max = 8.5, Time = 0.42 },
	Punch = { Reach = 9, Cooldown = 0.3, Lunge = 32, Radius = 2.4 },
	Dig = {
		Speed = 56, Descend = 0.95, Emerge = 0.7, MaxTime = 18, Drain = 5, MinEnergy = 25, TurnRate = 5,
		SafeFloor = 44, VoidMargin = 60, MaxStep = 5, Breach = 120,
	},
	Energy = { Max = 100, Regen = 16, RegenDelay = 0.9 },
	Break = { MaxDim = 34, Life = 11 },
	Repair = { Crater = 5.5, Mound = 3.2 },

	Camera = { Fov = 70, SpeedFov = 20, Zoom = { Min = 8, Max = 60 }, DigZoom = 28, Shake = 1.0 },
	ReduceMotion = false,
	AutoQuality = true,
	ShowButtonsOnDesktop = false,

	Colors = {
		Accent = Color3.fromRGB(255, 70, 55), Bone = Color3.fromRGB(226, 222, 210), Dark = Color3.fromRGB(12, 12, 14),
		Green = Color3.fromRGB(120, 255, 170), Cyan = Color3.fromRGB(110, 230, 255), Warn = Color3.fromRGB(255, 190, 60),
	},

	-- Sons: cole "rbxassetid://ID" (vazio = sem som, sem erro)
	Sounds = {
		Step = { Id = "", Volume = 0.8 }, Land = { Id = "", Volume = 1.1 }, Jump = { Id = "", Volume = 0.9 },
		Dash = { Id = "", Volume = 1.0 }, Punch = { Id = "", Volume = 1.1 }, Break = { Id = "", Volume = 1.2 },
		Dig = { Id = "", Volume = 1.2 }, Emerge = { Id = "", Volume = 1.2 }, Needle = { Id = "", Volume = 1.0 },
		Clamp = { Id = "", Volume = 1.2 }, WakeUp = { Id = "", Volume = 1.3 }, Servo = { Id = "", Volume = 0.8 },
		Grip = { Id = "", Volume = 0.9 }, Glitch = { Id = "", Volume = 0.9 },
	},
}

local PI = math.pi
local HALF_PI = PI / 2
local V3, CF, AN = Vector3.new, CFrame.new, CFrame.Angles
local IS_MOBILE = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
local DETAIL = IS_MOBILE and 1 or 2 -- 1 = menos peças/cabos/dentes (celular)

--------------------------------------------------------------------------------
-- UTILIDADES
--------------------------------------------------------------------------------
local U = {}
function U.clamp01(x) return x < 0 and 0 or (x > 1 and 1 or x) end
function U.smooth(x) x = U.clamp01(x) return x * x * (3 - 2 * x) end
function U.easeOut(x) x = U.clamp01(x) return 1 - (1 - x) ^ 3 end
function U.easeInOut(x)
	x = U.clamp01(x)
	if x < 0.5 then return 4 * x * x * x end
	return 1 - ((-2 * x + 2) ^ 3) / 2
end
function U.damp(rate, dt) return 1 - math.exp(-rate * dt) end
function U.wrap(a) return (a + PI) % (2 * PI) - PI end
function U.yawOf(v) return math.atan2(-v.X, -v.Z) end
function U.lerp(a, b, t) return a + (b - a) * t end
function U.hash01(s)
	local h = 7
	for i = 1, #s do h = (h * 31 + string.byte(s, i)) % 1000003 end
	return (h % 997) / 997
end
function U.bez(a, c, b, r) local k = 1 - r return a * (k * k) + c * (2 * k * r) + b * (r * r) end
-- rotação mínima que leva o vetor a até b
function U.rotFromTo(a, b)
	local c = a:Cross(b)
	local d = math.clamp(a:Dot(b), -1, 1)
	if c.Magnitude < 1e-5 then
		if d > 0 then return CFrame.new() end
		local perp = math.abs(a.X) < 0.9 and V3(1, 0, 0) or V3(0, 1, 0)
		return CFrame.fromAxisAngle(a:Cross(perp).Unit, PI)
	end
	return CFrame.fromAxisAngle(c.Unit, math.acos(d))
end
local clamp01, smooth, damp, wrap, yawOf, lerp = U.clamp01, U.smooth, U.damp, U.wrap, U.yawOf, U.lerp

-- raycast padrão (respeita CanCollide; ignora jogador, armadura e efeitos locais)
U.rp = RaycastParams.new()
U.rp.FilterType = Enum.RaycastFilterType.Exclude
U.rp.IgnoreWater = true
U.rp.RespectCanCollide = true
U.ignore = {}
function U.setIgnore(list) U.ignore = list; U.rp.FilterDescendantsInstances = list end
function U.ray(origin, dir) return Workspace:Raycast(origin, dir, U.rp) end

-- cor/material do chão atingido (funciona em terreno e em partes)
function U.groundLook(res)
	local dc, dm = Color3.fromRGB(112, 92, 72), Enum.Material.Ground
	if not res then return dc, dm end
	local inst = res.Instance
	if inst:IsA("Terrain") then
		local ok, c = pcall(function() return Workspace.Terrain:GetMaterialColor(res.Material) end)
		return ok and c or dc, res.Material
	end
	return inst.Color, inst.Material
end

local Actions = {} -- preenchido adiante (UI/entrada chamam estas funções)

--------------------------------------------------------------------------------
-- QUALIDADE ADAPTATIVA (mede o FPS e ajusta efeitos sozinha)
--------------------------------------------------------------------------------
local TIERS = {
	{ part = 0.25, shake = 0.45, blur = 0, lights = false, cables = 5, debris = 6 },
	{ part = 0.6, shake = 0.75, blur = 7, lights = true, cables = 8, debris = 12 },
	{ part = 1.0, shake = 1.0, blur = 12, lights = true, cables = 10, debris = 20 },
}
local tierIdx = IS_MOBILE and 2 or 3
local tierMax = IS_MOBILE and 2 or 3
do
	local ok, lvl = pcall(function() return settings().Rendering.QualityLevel.Value end)
	if ok and lvl > 0 then
		if lvl <= 4 then tierIdx = 1 elseif lvl <= 9 then tierIdx = math.min(tierIdx, 2) end
	end
	if CFG.ReduceMotion then tierIdx = 1 end
end
local function Q() return TIERS[tierIdx] end
do
	local ema, timer, since, good = 1 / 60, 0, 0, 0
	RunService.Heartbeat:Connect(function(dt)
		if not CFG.AutoQuality then return end
		dt = math.clamp(dt, 1 / 240, 0.25)
		ema += (dt - ema) * 0.05
		timer += dt
		since += dt
		if timer < 2 or since < 6 then return end
		timer = 0
		local fps = 1 / ema
		if fps < 30 and tierIdx > 1 then
			tierIdx -= 1; since = 0; good = 0
		elseif fps > 56 and tierIdx < tierMax then
			good += 2
			if good >= 14 then tierIdx += 1; since = 0; good = 0 end
		else
			good = 0
		end
	end)
end

--------------------------------------------------------------------------------
-- ESTADO GLOBAL
--------------------------------------------------------------------------------
local S = {
	ready = false, char = nil, hum = nil, root = nil, orig = nil, conns = {}, avatar = {},
	A = nil,              -- armadura (modelo, nós, efeitos...)
	mode = "Free",        -- Free | Cutscene | Armor
	mv = "ground",        -- ground | air | grip | mantle | dash | slam | burrow
	cut = nil,            -- cutscene ativa
	stealth = false, cloak = 0,
	energy = CFG.Energy.Max, lastDrain = -10,
	jumpHeld = false, jumpT0 = 0, charge = 0, coyote = 0, buffer = 0, airJumps = 1,
	gripHeld = false, wish = Vector3.zero, mvIn = Vector3.zero,
	dashCd = 0, punchCd = 0, combo = 0, punchT = -10, punchSide = 1,
	lastSafe = nil, nearArmor = false, status = "", statusT = 0, statusColor = nil,
	G = CFG.Body.G, spr = 0, sprV = 0, airT = 0, lastVy = 0, wasGround = true,
	phase = 0, spd = 0, vel = Vector3.zero, yawRate = 0, lastYaw = nil,
	wake = 0, ribOpen = 0, probeP = 0, neuralP = 0, jaw = 0, convulse = 0,
	burW = 0, gripW = 0, mantleW = 0, dashW = 0, crouchW = 0, airW = 0, drill = 0,
	tick = 0, eyeTarget = V3(0, 0, -1),
}

--------------------------------------------------------------------------------
-- EFEITOS: fábrica de emissores e âncora global de partículas
--------------------------------------------------------------------------------
local FXm = {}
local TEX_SPARK = "rbxasset://textures/particles/sparkles_main.dds"
local TEX_SMOKE = "rbxasset://textures/particles/smoke_main.dds"
local TEX_FIRE = "rbxasset://textures/particles/fire_main.dds"

local function nseq(a, b, c)
	if c then return NumberSequence.new({ NumberSequenceKeypoint.new(0, a), NumberSequenceKeypoint.new(0.5, b), NumberSequenceKeypoint.new(1, c) }) end
	return NumberSequence.new(a, b)
end
local function emitter(tex)
	local p = Instance.new("ParticleEmitter")
	p.Texture = tex; p.Enabled = false; p.Rate = 0; p.LockedToPart = false
	return p
end
function FXm.spark(col)
	local p = emitter(TEX_SPARK)
	p.Color = ColorSequence.new(col or Color3.fromRGB(255, 235, 190), Color3.fromRGB(255, 80, 40))
	p.LightEmission = 1; p.Size = nseq(0.5, 0.25, 0)
	p.Lifetime = NumberRange.new(0.25, 0.65); p.Speed = NumberRange.new(14, 38)
	p.SpreadAngle = Vector2.new(180, 180); p.Acceleration = V3(0, -60, 0); p.Drag = 2
	p.Transparency = nseq(0, 0.2, 1); p.Rotation = NumberRange.new(0, 360)
	return p
end
function FXm.dust()
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(150, 130, 105))
	p.Size = nseq(1.2, 3.6, 6); p.Lifetime = NumberRange.new(0.7, 1.4); p.Speed = NumberRange.new(8, 20)
	p.SpreadAngle = Vector2.new(75, 75); p.EmissionDirection = Enum.NormalId.Top; p.Drag = 3
	p.Transparency = nseq(0.45, 0.7, 1); p.Rotation = NumberRange.new(0, 360); p.RotSpeed = NumberRange.new(-40, 40)
	return p
end
function FXm.steam()
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(225, 230, 235))
	p.Size = nseq(0.4, 1.4, 2.4); p.Lifetime = NumberRange.new(0.7, 1.3); p.Speed = NumberRange.new(5, 11)
	p.SpreadAngle = Vector2.new(16, 16); p.EmissionDirection = Enum.NormalId.Top; p.Drag = 1.5
	p.Transparency = nseq(0.4, 0.7, 1); p.Rotation = NumberRange.new(0, 360)
	return p
end
function FXm.smoke()
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(40, 38, 38), Color3.fromRGB(100, 98, 96))
	p.Size = nseq(0.8, 2.6, 4.4); p.Lifetime = NumberRange.new(1.4, 2.4); p.Speed = NumberRange.new(3, 6)
	p.SpreadAngle = Vector2.new(10, 10); p.EmissionDirection = Enum.NormalId.Top
	p.Transparency = nseq(0.3, 0.6, 1); p.Rotation = NumberRange.new(0, 360)
	p:SetAttribute("BaseRate", 7)
	return p
end
function FXm.flame()
	local p = emitter(TEX_FIRE)
	p.Color = ColorSequence.new(Color3.fromRGB(255, 220, 160), Color3.fromRGB(255, 70, 30))
	p.LightEmission = 1; p.Size = nseq(1.4, 0.9, 0.1); p.Lifetime = NumberRange.new(0.2, 0.4)
	p.Speed = NumberRange.new(12, 22); p.SpreadAngle = Vector2.new(8, 8); p.EmissionDirection = Enum.NormalId.Bottom
	p.Transparency = nseq(0, 0.3, 1); p:SetAttribute("BaseRate", 70)
	return p
end
function FXm.burst(p, n)
	if p then p:Emit(math.max(1, math.floor(n * Q().part + 0.5))) end
end

-- peça invisível global que carrega emissores (movida para o ponto antes de Emit)
function FXm.init()
	local part = Instance.new("Part")
	part.Name = "PrypotFX"; part.Anchored = true; part.CanCollide = false; part.CanQuery = false; part.CanTouch = false
	part.Transparency = 1; part.Size = V3(1, 1, 1); part.Parent = Workspace
	local a = Instance.new("Attachment"); a.Parent = part
	FXm.part, FXm.att = part, a
	FXm.sparkE = FXm.spark(); FXm.sparkE.Parent = a
	FXm.redE = FXm.spark(Color3.fromRGB(255, 90, 60)); FXm.redE.Parent = a
	FXm.dustE = FXm.dust(); FXm.dustE.Parent = a
end
function FXm.at(pos, kind, n, color)
	if not FXm.part then return end
	FXm.part.CFrame = CF(pos)
	if kind == "dust" then
		if color then FXm.dustE.Color = ColorSequence.new(color) end
		FXm.burst(FXm.dustE, n)
	elseif kind == "red" then
		FXm.burst(FXm.redE, n)
	else
		FXm.burst(FXm.sparkE, n)
	end
end

--------------------------------------------------------------------------------
-- CÂMERA (FOV, zoom, offset, tremor, blur, flash) + modo cinematográfico
--------------------------------------------------------------------------------
local Cam = {
	trauma = 0, flash = 0, fov = 70, fovT = 70, off = Vector3.zero, offT = Vector3.zero, blurT = 0, tint = 0,
	zMin = 0.5, zMax = 128, zMinT = 0.5, zMaxT = 128, dMin = 0.5, dMax = 128, aMin = nil, aMax = nil,
	scripted = false, cutCF = nil, prevType = nil, vig = 0,
}
local blurFx = Instance.new("BlurEffect")
blurFx.Name = "PrypotBlur"; blurFx.Size = 0; blurFx.Enabled = false; blurFx.Parent = Lighting
local ccFx = Instance.new("ColorCorrectionEffect")
ccFx.Name = "PrypotGrade"; ccFx.Parent = Lighting

local function shake(a) Cam.trauma = math.min(1, Cam.trauma + a * (CFG.ReduceMotion and 0.15 or Q().shake) * CFG.Camera.Shake) end
local function flashScreen(a) Cam.flash = math.max(Cam.flash, a) end

function Cam.beginCut()
	local cam = Workspace.CurrentCamera
	if not cam or Cam.scripted then return end
	Cam.prevType = cam.CameraType
	Cam.scripted = true
	cam.CameraType = Enum.CameraType.Scriptable
end
function Cam.endCut()
	local cam = Workspace.CurrentCamera
	if cam and Cam.scripted then cam.CameraType = Cam.prevType or Enum.CameraType.Custom end
	Cam.scripted = false
end

function Cam.render(dt)
	dt = math.min(dt, 0.1)
	local cam = Workspace.CurrentCamera
	if not cam then return end
	if Cam.scripted and Cam.cutCF then cam.CFrame = Cam.cutCF end
	Cam.fov += (Cam.fovT - Cam.fov) * damp(5, dt)
	if math.abs(cam.FieldOfView - Cam.fov) > 0.01 then cam.FieldOfView = Cam.fov end

	if not Cam.scripted then
		Cam.zMin += (Cam.zMinT - Cam.zMin) * damp(2, dt)
		Cam.zMax += (Cam.zMaxT - Cam.zMax) * damp(2, dt)
		if not Cam.aMin or math.abs(Cam.zMin - Cam.aMin) > 0.05 or math.abs(Cam.zMax - Cam.aMax) > 0.05 then
			local nMin, nMax = Cam.zMin, math.max(Cam.zMax, Cam.zMin + 4)
			pcall(function()
				if nMax > player.CameraMaxZoomDistance then
					player.CameraMaxZoomDistance = nMax; player.CameraMinZoomDistance = nMin
				else
					player.CameraMinZoomDistance = nMin; player.CameraMaxZoomDistance = nMax
				end
			end)
			Cam.aMin, Cam.aMax = nMin, nMax
		end
		local hum = S.hum
		if hum and hum.Parent then
			Cam.off = Cam.off:Lerp(Cam.offT, damp(3, dt))
			if (hum.CameraOffset - Cam.off).Magnitude > 0.01 then hum.CameraOffset = Cam.off end
		end
		if Cam.trauma > 0 then
			Cam.trauma = math.max(0, Cam.trauma - dt * 1.5)
			local sh = Cam.trauma * Cam.trauma * 1.0
			local t = os.clock() * 28
			cam.CFrame = cam.CFrame * CF(math.noise(t, 0.3, 1.7) * sh * 0.35, math.noise(0.7, t, 7.3) * sh * 0.35, 0)
				* AN(0, 0, math.noise(t, t, 13.1) * sh * 0.02)
		end
	else
		Cam.trauma = math.max(0, Cam.trauma - dt * 1.5)
		if Cam.trauma > 0 then
			local sh = Cam.trauma * Cam.trauma
			local t = os.clock() * 30
			cam.CFrame = cam.CFrame * CF(math.noise(t, 0.3, 1.7) * sh * 0.25, math.noise(0.7, t, 7.3) * sh * 0.25, 0)
		end
	end

	local bt = (Q().blur > 0 and not CFG.ReduceMotion) and Cam.blurT * Q().blur or 0
	blurFx.Size += (bt - blurFx.Size) * damp(10, dt)
	blurFx.Enabled = blurFx.Size > 0.2
	Cam.flash = math.max(0, Cam.flash - dt * 2.2)
	ccFx.Brightness = Cam.flash * 0.4 - Cam.vig * 0.08
	ccFx.Contrast = Cam.flash * 0.25 + Cam.vig * 0.2
	ccFx.Saturation = -Cam.vig * 0.6 - S.cloak * 0.25
	ccFx.TintColor = Color3.new(1, 1 - Cam.tint * 0.45, 1 - Cam.tint * 0.5)
end

--------------------------------------------------------------------------------
-- DANOS NO CHÃO + REPARO AUTOMÁTICO  (nada do mapa é alterado: tudo é visual local)
--------------------------------------------------------------------------------
local Dmg = { list = {}, debris = {}, folder = nil }
local function maxActive() return IS_MOBILE and 16 or 30 end

function Dmg.init()
	Dmg.folder = Instance.new("Folder")
	Dmg.folder.Name = "PrypotDamage"
	Dmg.folder.Parent = Workspace
end

local function dmgPart(size, cf, color, mat, shape)
	local p = Instance.new("Part")
	p.Anchored = true; p.CanCollide = false; p.CanTouch = false; p.CanQuery = false; p.CastShadow = false
	p.Size = size; p.Color = color; p.Material = mat or Enum.Material.Ground; p.CFrame = cf
	p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
	if shape == "Cyl" then p.Shape = Enum.PartType.Cylinder end
	p.Parent = Dmg.folder
	return p
end

-- pedaços físicos que voam (vida curta; limitados)
function Dmg.spawnDebris(pos, dir, color, mat, count, speed, size)
	local cap = Q().debris
	count = math.min(count, cap)
	for _ = 1, count do
		if #Dmg.debris >= cap * 3 then
			local old = table.remove(Dmg.debris, 1)
			if old.part.Parent then old.part:Destroy() end
		end
		local p = Instance.new("Part")
		local s = size * (0.4 + math.random() * 0.8)
		p.Size = V3(s, s * (0.6 + math.random() * 0.6), s * (0.6 + math.random() * 0.6))
		p.Color = color; p.Material = mat; p.CFrame = CF(pos) * AN(math.random() * 6, math.random() * 6, math.random() * 6)
		p.CanCollide = true; p.CanTouch = false; p.CanQuery = false; p.CastShadow = false
		p.Parent = Dmg.folder
		local rnd = V3(math.random() - 0.5, math.random() * 0.8, math.random() - 0.5) * speed
		p.AssemblyLinearVelocity = dir * speed * (0.4 + math.random() * 0.6) + rnd
		p.AssemblyAngularVelocity = V3(math.random() - 0.5, math.random() - 0.5, math.random() - 0.5) * 14
		Dmg.debris[#Dmg.debris + 1] = { part = p, t0 = os.clock(), life = 2.2 + math.random() * 1.2 }
	end
end

local function trim()
	while #Dmg.list >= maxActive() do
		local e = table.remove(Dmg.list, 1)
		for _, p in ipairs(e.parts) do if p.Parent then p:Destroy() end end
	end
end

-- cratera: disco escuro + borda de terra + rachaduras brilhantes; SE REPARA no fim da vida
function Dmg.crater(pos, normal, radius, life, res)
	trim()
	local color, mat = U.groundLook(res)
	local dark = Color3.new(color.R * 0.4, color.G * 0.4, color.B * 0.4)
	local e = { t0 = os.clock(), life = life, parts = {}, radius = radius, normal = normal, kind = "crater", chunks = {} }
	local base = CFrame.lookAt(pos, pos + normal) * AN(0, HALF_PI, 0) -- eixo X do cilindro = normal
	e.disc = dmgPart(V3(0.2, radius * 2, radius * 2), base + normal * 0.05, dark, Enum.Material.Slate, "Cyl")
	e.ring = dmgPart(V3(0.1, radius * 2.3, radius * 2.3), base + normal * 0.08, Color3.fromRGB(255, 90, 40), Enum.Material.Neon, "Cyl")
	e.ring.Transparency = 0.35
	e.parts[1], e.parts[2] = e.disc, e.ring
	local t1 = normal:Cross(math.abs(normal.Y) > 0.9 and V3(1, 0, 0) or V3(0, 1, 0)).Unit
	local t2 = normal:Cross(t1)
	local n = DETAIL > 1 and 9 or 6
	for i = 1, n do
		local a = (i / n) * PI * 2 + math.random() * 0.4
		local d = (t1 * math.cos(a) + t2 * math.sin(a))
		local sz = radius * (0.28 + math.random() * 0.22)
		local cf = CFrame.lookAt(pos + d * radius * 1.0, pos + d * radius * 1.0 + normal) * AN(math.random() * 0.8, math.random() * 0.8, math.random() * 6)
		local c = dmgPart(V3(sz, sz * 0.8, sz), cf, color, mat)
		e.chunks[i] = { part = c, cf = cf }
		e.parts[#e.parts + 1] = c
	end
	e.upd = function(en, age)
		local grow = U.easeOut(age / 0.25)
		local rep = smooth((age - (en.life - 1.5)) / 1.5)
		local sc = math.max(0.02, grow * (1 - rep * 0.95))
		en.disc.Size = V3(0.2, en.radius * 2 * sc, en.radius * 2 * sc)
		en.ring.Size = V3(0.1, en.radius * 2.3 * sc, en.radius * 2.3 * sc)
		en.ring.Transparency = math.min(1, 0.35 + age / en.life * 0.7)
		for _, c in ipairs(en.chunks) do
			c.part.CFrame = c.cf - en.normal * (rep * en.radius * 0.5) - en.normal * ((1 - grow) * 0.6)
			c.part.Transparency = rep
		end
	end
	e.pos, e.color = pos, color
	Dmg.list[#Dmg.list + 1] = e
	FXm.at(pos, "dust", math.floor(radius * 7), color)
	Dmg.spawnDebris(pos + normal * 0.5, normal, color, mat, math.floor(radius * 3), radius * 14, radius * 0.25)
end

-- montinho de terra levantado no rastro de escavação
function Dmg.mound(pos, dir, size, life, res)
	trim()
	local color, mat = U.groundLook(res)
	local e = { t0 = os.clock(), life = life, parts = {}, kind = "mound" }
	local cf = CFrame.lookAt(pos, pos + dir)
	e.p = dmgPart(V3(size, size * 0.35, size * 0.9), cf, color, mat)
	e.cf, e.size = cf, size
	e.parts[1] = e.p
	e.upd = function(en, age)
		local rise = U.easeOut(age / 0.22)
		local rep = smooth((age - (en.life - 0.9)) / 0.9)
		local k = rise * (1 - rep)
		en.p.Size = V3(en.size * (0.6 + 0.4 * k), en.size * 0.35 * k + 0.05, en.size * 0.9)
		en.p.CFrame = en.cf * CF(0, en.size * 0.12 * k, 0) * AN(0.2 * (1 - k), 0, 0)
		en.p.Transparency = rep * 0.9
	end
	Dmg.list[#Dmg.list + 1] = e
end

-- rachaduras na parede/solo (soco que não quebra)
function Dmg.cracks(pos, normal, dir, res)
	trim()
	local color, mat = U.groundLook(res)
	local e = { t0 = os.clock(), life = 5, parts = {}, kind = "cracks", items = {} }
	local t1 = normal:Cross(math.abs(normal.Y) > 0.9 and V3(1, 0, 0) or V3(0, 1, 0)).Unit
	local t2 = normal:Cross(t1)
	local n = DETAIL > 1 and 7 or 5
	for i = 1, n do
		local a = (i / n) * PI * 2 + math.random() * 0.5
		local d = (t1 * math.cos(a) + t2 * math.sin(a))
		local len = 2 + math.random() * 3.2
		local cf = CFrame.lookAt(pos + d * len * 0.5 + normal * 0.04, pos + d * len + normal * 0.04)
		local p = dmgPart(V3(0.16, 0.06, len), cf, Color3.fromRGB(12, 10, 10), Enum.Material.SmoothPlastic)
		e.parts[#e.parts + 1] = p
		e.items[#e.items + 1] = p
	end
	e.upd = function(en, age)
		local f = smooth((age - 3.6) / 1.4)
		for _, p in ipairs(en.items) do p.Transparency = f end
	end
	Dmg.list[#Dmg.list + 1] = e
	FXm.at(pos + normal * 0.2, "spark", 10)
	Dmg.spawnDebris(pos + normal * 0.5, normal + dir * 0.4, color, mat, 5, 22, 0.5)
end

function Dmg.update()
	local now = os.clock()
	for i = #Dmg.list, 1, -1 do
		local e = Dmg.list[i]
		local age = now - e.t0
		if age >= e.life then
			if e.kind == "crater" then FXm.at(e.pos, "dust", 6, e.color) end
			for _, p in ipairs(e.parts) do if p.Parent then p:Destroy() end end
			table.remove(Dmg.list, i)
		else
			e.upd(e, age)
		end
	end
	for i = #Dmg.debris, 1, -1 do
		local d = Dmg.debris[i]
		local age = now - d.t0
		if not d.part.Parent then
			table.remove(Dmg.debris, i)
		elseif age >= d.life then
			d.part:Destroy()
			table.remove(Dmg.debris, i)
		elseif age > d.life - 0.7 then
			d.part.Transparency = (age - (d.life - 0.7)) / 0.7
		end
	end
end

--------------------------------------------------------------------------------
-- QUEBRAR PAREDES / OBJETOS COM SOCOS (visual apenas para o jogador local)
--------------------------------------------------------------------------------
local Brk = { broken = {} }

function Brk.canBreak(part)
	if not part or not part:IsA("BasePart") or part:IsA("Terrain") then return false end
	if S.char and part:IsDescendantOf(S.char) then return false end
	if S.A and S.A.model and part:IsDescendantOf(S.A.model) then return false end
	if Dmg.folder and part:IsDescendantOf(Dmg.folder) then return false end
	local model = part:FindFirstAncestorOfClass("Model")
	if model and Players:GetPlayerFromCharacter(model) then return false end
	if CollectionService:HasTag(part, "NoBreak") or Brk.broken[part] then return false end
	return true
end

function Brk.hide(part, on)
	local st = Brk.broken[part]
	if on then
		st = { cc = part.CanCollide, ct = part.CanTouch, decals = {} }
		Brk.broken[part] = st
		part.CanCollide = false
		part.CanTouch = false
		part.LocalTransparencyModifier = 1
		for _, d in ipairs(part:GetDescendants()) do
			if d:IsA("Decal") then d.LocalTransparencyModifier = 1; st.decals[#st.decals + 1] = d end
		end
	elseif st then
		part.CanCollide = st.cc
		part.CanTouch = st.ct
		part.LocalTransparencyModifier = 0
		for _, d in ipairs(st.decals) do if d.Parent then d.LocalTransparencyModifier = 0 end end
		Brk.broken[part] = nil
	end
end

function Brk.shatter(part, hitPos, dir)
	local size = part.Size
	local maxChunks = IS_MOBILE and 10 or 18
	local nx = math.clamp(math.ceil(size.X / 5), 1, 3)
	local ny = math.clamp(math.ceil(size.Y / 5), 1, 3)
	local nz = math.clamp(math.ceil(size.Z / 5), 1, 3)
	while nx * ny * nz > maxChunks do
		if nx >= ny and nx >= nz and nx > 1 then nx -= 1 elseif ny >= nz and ny > 1 then ny -= 1 else nz = math.max(1, nz - 1) end
	end
	local cs = V3(size.X / nx, size.Y / ny, size.Z / nz)
	local cf0, color, mat = part.CFrame, part.Color, part.Material
	local cap = Q().debris * 2
	Brk.hide(part, true)
	local made = 0
	for ix = 0, nx - 1 do for iy = 0, ny - 1 do for iz = 0, nz - 1 do
		if made < cap then
			made += 1
			local off = V3((ix + 0.5) * cs.X - size.X / 2, (iy + 0.5) * cs.Y - size.Y / 2, (iz + 0.5) * cs.Z - size.Z / 2)
			local c = Instance.new("Part")
			c.Size = cs * 0.96; c.Color = color; c.Material = mat; c.CFrame = cf0 * CF(off)
			c.CanCollide = true; c.CanTouch = false; c.CanQuery = false; c.CastShadow = false
			c.Parent = Dmg.folder
			local wpos = c.Position
			local away = (wpos - hitPos)
			away = away.Magnitude > 0.1 and away.Unit or dir
			c.AssemblyLinearVelocity = (dir * 38 + away * 22) * (0.6 + math.random() * 0.7) + V3(0, 12, 0)
			c.AssemblyAngularVelocity = V3(math.random() - 0.5, math.random() - 0.5, math.random() - 0.5) * 12
			Dmg.debris[#Dmg.debris + 1] = { part = c, t0 = os.clock(), life = 3 + math.random() * 1.5 }
		end
	end end end
	FXm.at(hitPos, "dust", 18, color)
	FXm.at(hitPos, "spark", 12)
	shake(0.5)
	task.delay(CFG.Break.Life, function()
		if part.Parent and Brk.broken[part] then
			Brk.hide(part, false)           -- a parede se reconstrói
			FXm.at(part.Position, "spark", 8)
		end
	end)
end

function Brk.hit(res, dir)
	local part = res.Instance
	if not Brk.canBreak(part) then return false end
	local s = part.Size
	if math.max(s.X, s.Y, s.Z) <= CFG.Break.MaxDim then
		Brk.shatter(part, res.Position, dir)
		return true
	end
	Dmg.cracks(res.Position, res.Normal, dir, res) -- parte enorme: só rachaduras
	return false
end

--------------------------------------------------------------------------------
-- SONS (opcionais; sem ID = silêncio, sem erro)
--------------------------------------------------------------------------------
local Sfx = {}
function Sfx.play(key, pv)
	if S.stealth and (key == "Step" or key == "Land") then return end
	local A = S.A
	local snd = A and A.sounds[key]
	if not snd then return end
	snd.PlaybackSpeed = 1 + (math.random() - 0.5) * (pv or 0)
	snd.TimePosition = 0
	snd:Play()
end

--------------------------------------------------------------------------------
-- INTERFACE MOBILE
--------------------------------------------------------------------------------
local UI = { b = {}, built = false, statusFade = 0 }

local function corner(p, r) local c = Instance.new("UICorner") c.CornerRadius = r c.Parent = p return c end
local function stroke(p, color, th, tr)
	local s = Instance.new("UIStroke")
	s.Color = color; s.Thickness = th; s.Transparency = tr or 0
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border; s.Parent = p
	return s
end
local function label(parent, font, color, xa)
	local l = Instance.new("TextLabel")
	l.BackgroundTransparency = 1; l.Font = font; l.TextColor3 = color; l.TextScaled = true
	l.TextXAlignment = xa or Enum.TextXAlignment.Center; l.Parent = parent
	return l
end

function UI.makeButton(key, text, color)
	local b = Instance.new("TextButton")
	b.Name = "Btn_" .. key
	b.AutoButtonColor = false; b.Selectable = false; b.Active = true
	b.AnchorPoint = Vector2.new(1, 1)
	b.BackgroundColor3 = CFG.Colors.Dark; b.BackgroundTransparency = 0.25
	b.Text = ""; b.Visible = false; b.Parent = UI.gui
	corner(b, UDim.new(1, 0))
	local st = stroke(b, color, 3, 0.05)
	local ring = Instance.new("Frame")
	ring.AnchorPoint = Vector2.new(0.5, 0.5); ring.Position = UDim2.fromScale(0.5, 0.5)
	ring.Size = UDim2.fromScale(0.8, 0.8); ring.BackgroundTransparency = 1; ring.Parent = b
	corner(ring, UDim.new(1, 0)); stroke(ring, color, 1, 0.65)
	local l = label(b, Enum.Font.GothamBlack, Color3.new(1, 1, 1))
	l.AnchorPoint = Vector2.new(0.5, 0.5); l.Position = UDim2.fromScale(0.5, 0.5); l.Size = UDim2.fromScale(0.78, 0.3)
	l.Text = text
	local sc = Instance.new("UIScale"); sc.Parent = b
	local function press(d) TweenService:Create(sc, TweenInfo.new(0.07, Enum.EasingStyle.Quad), { Scale = d and 0.88 or 1 }):Play() end
	b.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then press(true) end end)
	b.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then press(false) end end)
	local rec = { f = b, label = l, stroke = st, color = color, on = true, text = text }
	UI.b[key] = rec
	return rec
end

-- botão "segurar": rastreia o toque exato (solta mesmo se o dedo sair do botão)
function UI.hold(rec, fn)
	local active
	rec.f.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then active = i fn(true) end
	end)
	UserInputService.InputEnded:Connect(function(i) if active and i == active then active = nil fn(false) end end)
end

function UI.bar(color)
	local f = Instance.new("Frame")
	f.BackgroundColor3 = CFG.Colors.Dark; f.BackgroundTransparency = 0.3; f.Parent = UI.gui
	corner(f, UDim.new(1, 0)); stroke(f, color, 2, 0.35)
	local fill = Instance.new("Frame")
	fill.Size = UDim2.fromScale(1, 1); fill.BackgroundColor3 = color; fill.BorderSizePixel = 0; fill.Parent = f
	corner(fill, UDim.new(1, 0))
	return f, fill
end

function UI.build()
	if UI.built then return end
	UI.built = true
	local gui = Instance.new("ScreenGui")
	gui.Name = "PrypotUI"; gui.ResetOnSpawn = false; gui.IgnoreGuiInset = true
	gui.DisplayOrder = 60; gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	pcall(function() gui.ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets end)
	gui.Parent = player:WaitForChild("PlayerGui")
	UI.gui = gui

	local C = CFG.Colors
	local jump = UI.makeButton("Jump", "PULO", C.Cyan)
	local punch = UI.makeButton("Punch", "SOCO", C.Accent)
	local dash = UI.makeButton("Dash", "DASH", C.Warn)
	local grip = UI.makeButton("Grip", "GARRA", C.Green)
	local dig = UI.makeButton("Dig", "ESCAVAR", Color3.fromRGB(200, 140, 90))
	local stealth = UI.makeButton("Stealth", "FURTIVO", Color3.fromRGB(170, 120, 255))
	local act = UI.makeButton("Action", "ENTRAR", C.Bone)
	local skip = UI.makeButton("Skip", "PULAR", Color3.fromRGB(200, 200, 200))
	UI.hold(jump, function(on) Actions.jump(on) end)
	UI.hold(grip, function(on) Actions.grip(on) end)
	punch.f.Activated:Connect(function() Actions.punch() end)
	dash.f.Activated:Connect(function() Actions.dash() end)
	dig.f.Activated:Connect(function() Actions.dig() end)
	stealth.f.Activated:Connect(function() Actions.stealth() end)
	act.f.Activated:Connect(function() Actions.interact() end)
	skip.f.Activated:Connect(function() Actions.skip() end)

	local eb, ef = UI.bar(C.Green)
	eb.AnchorPoint = Vector2.new(0.5, 0)
	UI.energy, UI.energyFill = eb, ef
	local cb, cf = UI.bar(C.Cyan)
	cb.AnchorPoint = Vector2.new(0.5, 1)
	UI.charge, UI.chargeFill = cb, cf

	UI.status = label(gui, Enum.Font.GothamBold, C.Warn)
	UI.status.AnchorPoint = Vector2.new(0.5, 0); UI.status.TextStrokeTransparency = 0.4; UI.status.Visible = false
	UI.tip = label(gui, Enum.Font.GothamMedium, Color3.fromRGB(235, 230, 225))
	UI.tip.AnchorPoint = Vector2.new(0.5, 1); UI.tip.TextScaled = false; UI.tip.TextStrokeTransparency = 0.5

	-- cinema: barras, estática, título
	UI.barTop = Instance.new("Frame"); UI.barTop.BackgroundColor3 = Color3.new(0, 0, 0); UI.barTop.BorderSizePixel = 0
	UI.barTop.Size = UDim2.new(1, 0, 0, 0); UI.barTop.Parent = gui
	UI.barBot = Instance.new("Frame"); UI.barBot.BackgroundColor3 = Color3.new(0, 0, 0); UI.barBot.BorderSizePixel = 0
	UI.barBot.AnchorPoint = Vector2.new(0, 1); UI.barBot.Position = UDim2.fromScale(0, 1); UI.barBot.Size = UDim2.new(1, 0, 0, 0)
	UI.barBot.Parent = gui
	UI.noise = Instance.new("Frame"); UI.noise.Size = UDim2.fromScale(1, 1); UI.noise.BackgroundTransparency = 1
	UI.noise.BorderSizePixel = 0; UI.noise.Parent = gui
	UI.noiseBars = {}
	for i = 1, 7 do
		local f = Instance.new("Frame")
		f.BorderSizePixel = 0; f.BackgroundColor3 = i % 2 == 0 and Color3.fromRGB(255, 60, 50) or Color3.new(1, 1, 1)
		f.BackgroundTransparency = 1; f.Size = UDim2.new(1, 0, 0, 3); f.Parent = UI.noise
		UI.noiseBars[i] = f
	end
	UI.flashF = Instance.new("Frame"); UI.flashF.Size = UDim2.fromScale(1, 1); UI.flashF.BackgroundColor3 = Color3.fromRGB(255, 40, 30)
	UI.flashF.BackgroundTransparency = 1; UI.flashF.BorderSizePixel = 0; UI.flashF.Parent = gui
	UI.title = label(gui, Enum.Font.GothamBlack, C.Accent)
	UI.title.AnchorPoint = Vector2.new(0.5, 0.5); UI.title.Position = UDim2.fromScale(0.5, 0.42); UI.title.Size = UDim2.fromScale(0.7, 0.1)
	UI.title.TextTransparency = 1; UI.title.TextStrokeTransparency = 1
	UI.sub = label(gui, Enum.Font.GothamMedium, Color3.new(1, 1, 1))
	UI.sub.AnchorPoint = Vector2.new(0.5, 0.5); UI.sub.Position = UDim2.fromScale(0.5, 0.5); UI.sub.Size = UDim2.fromScale(0.6, 0.045)
	UI.sub.TextTransparency = 1
end

function UI.layout()
	if not UI.built then return end
	local cam = Workspace.CurrentCamera
	local vp = cam and cam.ViewportSize or Vector2.new(800, 400)
	local s = math.clamp(math.min(vp.X, vp.Y) * 0.15, 52, 84)
	local m, B = s * 0.22, UI.b
	local function place(key, size, dx, dy)
		B[key].f.Size = UDim2.fromOffset(size, size)
		B[key].f.AnchorPoint = Vector2.new(1, 1)
		B[key].f.Position = UDim2.new(1, -dx, 1, -dy)
	end
	place("Jump", s * 1.22, m, m)
	place("Punch", s * 1.1, m + s * 1.38, m + s * 0.05)
	place("Dash", s, m + s * 0.1, m + s * 1.4)
	place("Grip", s, m + s * 1.28, m + s * 1.5)
	place("Dig", s * 0.95, m + s * 2.5, m + s * 0.1)
	place("Stealth", s * 0.9, m + s * 0.15, m + s * 2.5)
	B.Action.f.Size = UDim2.fromOffset(s * 1.05, s * 1.05)
	B.Action.f.AnchorPoint = Vector2.new(1, 0)
	B.Action.f.Position = UDim2.new(1, -m, 0, vp.Y * 0.14)
	B.Skip.f.Size = UDim2.fromOffset(s * 0.95, s * 0.95)
	B.Skip.f.AnchorPoint = Vector2.new(0.5, 1)
	B.Skip.f.Position = UDim2.new(0.5, 0, 1, -m * 1.4)
	local pw = math.clamp(vp.X * 0.4, 160, 320)
	UI.energy.Size = UDim2.fromOffset(pw, 8); UI.energy.Position = UDim2.new(0.5, 0, 0, 12)
	UI.charge.Size = UDim2.fromOffset(pw * 0.6, 7); UI.charge.Position = UDim2.new(0.5, 0, 1, -(m + 6))
	UI.status.Size = UDim2.fromOffset(vp.X * 0.7, math.clamp(vp.Y * 0.05, 16, 26)); UI.status.Position = UDim2.new(0.5, 0, 0, 28)
	UI.tip.Size = UDim2.fromOffset(vp.X * 0.8, 22); UI.tip.TextSize = math.clamp(vp.Y * 0.034, 12, 17)
	UI.tip.Position = UDim2.new(0.5, 0, 1, -6)
end

local lastInput = UserInputService:GetLastInputType()
UserInputService.LastInputTypeChanged:Connect(function(t) lastInput = t end)
function UI.wantButtons()
	if CFG.ShowButtonsOnDesktop or lastInput == Enum.UserInputType.Touch then return true end
	return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled and not UserInputService.GamepadEnabled
end

function UI.setB(key, show, enabled, text)
	local r = UI.b[key]
	r.f.Visible = show
	if text and r.text ~= text then r.text = text r.label.Text = text end
	if r.on ~= enabled then
		r.on = enabled
		r.f.BackgroundTransparency = enabled and 0.25 or 0.7
		r.label.TextTransparency = enabled and 0 or 0.55
		r.stroke.Transparency = enabled and 0.05 or 0.65
	end
end

function UI.say(text, color, secs)
	S.status, S.statusT, S.statusColor = text, os.clock() + (secs or 2.5), color or CFG.Colors.Warn
end

-- esconde o botão de pulo padrão do Roblox (usamos o nosso, que carrega o superpulo)
function UI.hideDefaultJump(hide)
	pcall(function()
		local tg = player.PlayerGui:FindFirstChild("TouchGui")
		local jb = tg and tg:FindFirstChild("TouchControlFrame") and tg.TouchControlFrame:FindFirstChild("JumpButton")
		if jb then jb.Visible = not hide end
	end)
end

function UI.update(t, dt)
	if not UI.built then return end
	local touch = UI.wantButtons()
	local inArmor = S.mode == "Armor"
	local cut = S.mode == "Cutscene"
	local grounded = S.mv == "ground"
	local B = UI.b

	UI.setB("Jump", touch and inArmor, true, nil)
	UI.setB("Punch", touch and inArmor, S.punchCd <= 0, nil)
	UI.setB("Dash", touch and inArmor, S.energy >= CFG.Dash.Cost and S.dashCd <= 0, nil)
	UI.setB("Grip", touch and inArmor, S.energy > 5, nil)
	UI.setB("Dig", touch and inArmor, S.energy >= 5, S.mv == "burrow" and "SURGIR" or "ESCAVAR")
	UI.setB("Stealth", touch and inArmor, true, S.stealth and "REVELAR" or "FURTIVO")
	local showAct = S.ready and ((S.mode == "Free" and S.nearArmor) or (inArmor and grounded))
	UI.setB("Action", touch and showAct, true, inArmor and "SAIR" or "ENTRAR")
	UI.setB("Skip", cut, true, nil)

	UI.energy.Visible = inArmor
	UI.charge.Visible = inArmor and S.charge > 0.02
	if inArmor then
		UI.energyFill.Size = UDim2.fromScale(clamp01(S.energy / CFG.Energy.Max), 1)
		UI.energyFill.BackgroundColor3 = S.energy < 20 and CFG.Colors.Accent or CFG.Colors.Green
		UI.chargeFill.Size = UDim2.fromScale(clamp01(S.charge), 1)
		UI.chargeFill.BackgroundColor3 = S.charge >= 0.98 and CFG.Colors.Accent or CFG.Colors.Cyan
	end

	local hasStatus = t < S.statusT
	UI.status.Visible = hasStatus
	if hasStatus then UI.status.Text = S.status; UI.status.TextColor3 = S.statusColor or CFG.Colors.Warn end

	-- dicas (PC)
	if S.mode == "Free" and not touch then
		UI.tip.Text = "Aproxime-se da armadura  •  E: entrar"; UI.tip.Visible = true
	elseif inArmor and not touch then
		UI.tip.Text = "ESPAÇO pulo (segure)  •  F soco  •  SHIFT dash  •  Q garra  •  R escavar  •  C furtivo  •  E sair"
		UI.tip.Visible = true
	else
		UI.tip.Visible = false
	end

	-- cinema (barras + estática + título)
	local cutK = S.cut and S.cut.letter or 0
	local vh = (Workspace.CurrentCamera and Workspace.CurrentCamera.ViewportSize.Y or 400)
	UI.barTop.Size = UDim2.new(1, 0, 0, vh * 0.11 * cutK)
	UI.barBot.Size = UDim2.new(1, 0, 0, vh * 0.11 * cutK)
	local glitch = S.cut and S.cut.glitch or 0
	for i, f in ipairs(UI.noiseBars) do
		if glitch > 0.02 and math.random() < 0.55 then
			f.BackgroundTransparency = 1 - glitch * (0.25 + math.random() * 0.5)
			f.Position = UDim2.fromScale(0, math.random())
			f.Size = UDim2.new(1, 0, 0, 1 + math.random() * 5)
		else
			f.BackgroundTransparency = 1
		end
	end
	UI.flashF.BackgroundTransparency = 1 - math.min(0.6, (S.cut and S.cut.red or 0))
	UI.title.TextTransparency = 1 - (S.cut and S.cut.title or 0)
	UI.title.TextStrokeTransparency = 1 - (S.cut and S.cut.title or 0) * 0.6
	UI.sub.TextTransparency = 1 - (S.cut and S.cut.sub or 0)

	UI.tick = (UI.tick or 0) + 1
	if UI.tick % 30 == 0 then UI.hideDefaultJump(inArmor and touch) end
end

--------------------------------------------------------------------------------
-- MODELO DA ARMADURA (procedural). Espaço: origem no CHÃO sob a pelve, frente = -Z
-- Pelve de repouso em y = 5.0 (a pose real é definida por IK em tempo real)
--------------------------------------------------------------------------------
local Bld = {}
local BONE, BONE_D = Color3.fromRGB(226, 222, 210), Color3.fromRGB(190, 184, 170)
local BLACK, STEEL, STEEL_D = Color3.fromRGB(18, 18, 20), Color3.fromRGB(96, 100, 108), Color3.fromRGB(52, 54, 60)
local OLIVE, OLIVE_D = Color3.fromRGB(84, 92, 62), Color3.fromRGB(56, 62, 44)
local BRASS, RUSTC, REDC = Color3.fromRGB(176, 136, 62), Color3.fromRGB(110, 58, 36), Color3.fromRGB(255, 40, 30)
local WHITE, YEL = Color3.fromRGB(255, 255, 255), Color3.fromRGB(232, 182, 30)
local METAL, SMOOTH, NEON, DIAMOND = Enum.Material.Metal, Enum.Material.SmoothPlastic, Enum.Material.Neon, Enum.Material.DiamondPlate
local cylY, cylZ = AN(0, 0, HALF_PI), AN(0, HALF_PI, 0)
local HIPX, ANKLE_H, L1, L2 = 1.3, 0.6, 2.0, 2.0

local function lk(a, b)
	local d = b - a
	if d.Magnitude < 1e-4 then return CF(a) end
	local up = math.abs(d.Unit.Y) > 0.98 and V3(0, 0, -1) or V3(0, 1, 0)
	return CFrame.lookAt((a + b) * 0.5, b, up)
end

function Bld.build(O)
	local model = Instance.new("Model")
	model.Name = "PrypotArmor"
	model.Parent = Workspace
	local N, list, parts = {}, {}, {}
	local A = {
		model = model, N = N, list = list, parts = parts, sounds = {}, drill = {}, solids = {}, neural = {}, leds = {}, slashes = {},
		pistons = {}, springs = {}, cables = {}, aux = {}, probes = {}, pupils = {}, vents = {}, stacks = {}, glowAll = {},
	}
	local count = 0

	local function tint(c, name)
		local k = 1 + (U.hash01(name) - 0.5) * 0.16
		return Color3.new(math.clamp(c.R * k, 0, 1), math.clamp(c.G * k, 0, 1), math.clamp(c.B * k, 0, 1))
	end
	local function mk(name, shape, size, cf, color, mat)
		local p = Instance.new("Part")
		p.Name = name; p.Size = size; p.Anchored = true
		p.CanCollide = false; p.CanTouch = false; p.Massless = true
		p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
		p.Color = (mat == NEON) and color or tint(color, name)
		p.Material = mat or METAL
		p.CastShadow = math.max(size.X, size.Y, size.Z) > 1.6
		if shape == "Cyl" then p.Shape = Enum.PartType.Cylinder elseif shape == "Ball" then p.Shape = Enum.PartType.Ball end
		p.CFrame = O * cf
		p.Parent = model
		parts[#parts + 1] = p
		return p
	end
	local function node(name, parentName, shape, size, rest, joint, color, mat, solid)
		local p = mk(name, shape, size, rest, color, mat)
		local n = { name = name, part = p, rest = rest, solid = solid, tx = 0, ty = 0, tz = 0, cx = 0, cy = 0, cz = 0, ax = 1e9, ay = 0, az = 0 }
		if parentName then
			local par = N[parentName]
			local m = Instance.new("Motor6D")
			m.Name = name .. "Joint"; m.Part0 = par.part; m.Part1 = p
			m.C0 = par.rest:Inverse() * CF(joint); m.C1 = rest:Inverse() * CF(joint)
			m.Parent = p
			n.motor, n.c0, n.joint = m, m.C0, joint
			n.parent = par
		end
		N[name] = n; list[#list + 1] = n
		return n
	end
	-- osso entre dois pontos (junta no ponto a)
	local function bnode(name, parentName, a, b, thick, color, mat, solid)
		return node(name, parentName, "Block", V3(thick, thick, (b - a).Magnitude), lk(a, b), a, color, mat, solid)
	end
	local function detW(nodeName, shape, size, worldCF, color, mat)
		local n = N[nodeName]
		count += 1
		local p = mk(nodeName .. "_" .. count, shape, size, worldCF, color, mat)
		local w = Instance.new("WeldConstraint"); w.Part0, w.Part1, w.Parent = n.part, p, p
		if mat == NEON then A.glowAll[#A.glowAll + 1] = p end
		return p
	end
	local function bar(nodeName, a, b, thick, color, mat)
		local d = (b - a).Magnitude
		return detW(nodeName, "Block", V3(thick, thick, d), lk(a, b), color, mat)
	end
	local function att(nodeName, name, worldPos)
		local a = Instance.new("Attachment")
		a.Name = name; a.Position = N[nodeName].rest:PointToObjectSpace(worldPos); a.Parent = N[nodeName].part
		return a
	end
	local function anchorW(nodeName, worldPos) return { n = N[nodeName], off = N[nodeName].rest:PointToObjectSpace(worldPos) } end
	local function gear(parentName, name, center, axis, R, thick, rate)
		local rot = axis == "y" and cylY or (axis == "z" and cylZ or CFrame.new())
		local g = node(name, parentName, "Cyl", V3(thick, R * 2, R * 2), CF(center) * rot, center, BRASS, METAL)
		g.spinRate, g.spinAxis, g.ang = rate, axis, U.hash01(name) * PI
		local teeth = DETAIL > 1 and 8 or 6
		for i = 0, teeth - 1 do
			detW(name, "Block", V3(thick * 0.9, R * 0.28, R * 0.32), g.rest * AN(i * 2 * PI / teeth, 0, 0) * CF(0, R * 1.08, 0), RUSTC)
		end
		detW(name, "Cyl", V3(thick * 1.4, R * 0.5, R * 0.5), g.rest, STEEL_D)
		detW(name, "Block", V3(thick * 0.5, R * 1.7, R * 0.15), g.rest, STEEL_D)
		return g
	end

	-- PELVE ---------------------------------------------------------------------
	node("Pelvis", nil, "Block", V3(2.8, 1.2, 1.7), CF(0, 5.0, 0), nil, STEEL_D, METAL, true)
	for _, sx in ipairs({ -1, 1 }) do
		detW("Pelvis", "Block", V3(0.5, 1.0, 1.4), CF(sx * 1.55, 5.05, 0) * AN(0, 0, sx * 0.25), OLIVE)
		for i = 0, 2 do detW("Pelvis", "Block", V3(0.3, 0.18, 0.08), CF(sx * 1.0 + (i - 1) * 0.35, 4.55, -0.9), YEL) end
		gear("Pelvis", "GearHip" .. (sx < 0 and "L" or "R"), V3(sx * 2.05, 5.0, 0), "x", 0.7, 0.28, 1.0 * sx)
	end
	detW("Pelvis", "Block", V3(1.3, 0.9, 0.3), CF(0, 4.7, -0.95), OLIVE)
	detW("Pelvis", "Block", V3(1.7, 0.8, 0.3), CF(0, 5.0, 0.95), STEEL)
	detW("Pelvis", "Block", V3(3.0, 0.28, 1.95), CF(0, 5.45, 0), BLACK, SMOOTH)
	detW("Pelvis", "Block", V3(0.4, 0.25, 0.1), CF(0, 5.45, -1.0), REDC, NEON)
	local hull = detW("Pelvis", "Block", V3(3.4, 9.4, 2.6), CF(0, 5.2, 0.1), STEEL)
	hull.Transparency = 1; hull.Name = "Hull"; hull.CastShadow = false
	A.hull = hull
	parts[#parts] = nil -- o casco invisível não entra na lista de "capa" (furtivo)

	-- COLUNA / TRONCO -----------------------------------------------------------
	node("Torso", "Pelvis", "Block", V3(0.7, 3.0, 0.7), CF(0, 7.0, 0.5), V3(0, 5.5, 0.5), BONE, SMOOTH, true)
	for i = 0, 5 do
		local y = 5.65 + i * 0.5
		detW("Torso", "Block", V3(0.95, 0.34, 0.95), CF(0, y, 0.5), i % 2 == 0 and BONE or BLACK, SMOOTH)
		detW("Torso", "Block", V3(0.2, 0.3, 0.3), CF(0, y, 1.05), BONE_D, SMOOTH)
		A.neural[#A.neural + 1] = detW("Torso", "Block", V3(0.28, 0.18, 0.12), CF(0, y, 1.18), Color3.fromRGB(70, 10, 8), NEON)
	end
	detW("Torso", "Block", V3(4.3, 0.3, 0.45), CF(0, 8.35, -0.1), BONE, SMOOTH)     -- clavículas
	bar("Torso", V3(0, 8.4, 0.3), V3(0, 8.95, 0.1), 0.34, BLACK, METAL)             -- pescoço
	detW("Torso", "Cyl", V3(0.3, 0.55, 0.55), CF(0, 8.55, 0.25) * cylY, STEEL_D)
	for _, sx in ipairs({ -1, 1 }) do detW("Torso", "Cyl", V3(0.5, 0.9, 0.9), CF(sx * 2.0, 8.1, 0), BLACK, METAL) end
	-- mochila / motor / baia neural
	detW("Torso", "Block", V3(1.9, 1.4, 0.8), CF(0, 7.95, 1.5), OLIVE_D)
	detW("Torso", "Block", V3(1.1, 1.3, 0.32), CF(0, 6.55, 1.12), BLACK, METAL)    -- baia das agulhas
	for i = 0, 4 do detW("Torso", "Cyl", V3(0.12, 0.16, 0.16), CF(-0.4 + i * 0.2, 6.55 + (i % 2) * 0.2 - 0.1, 1.3) * cylZ, STEEL) end
	for _, sx in ipairs({ -1, 1 }) do
		detW("Torso", "Cyl", V3(1.5, 0.55, 0.55), CF(sx * 0.62, 8.95, 1.5) * cylY, STEEL)
		detW("Torso", "Cyl", V3(0.2, 0.75, 0.75), CF(sx * 0.62, 9.72, 1.5) * cylY, BLACK)
		A.stacks[#A.stacks + 1] = att("Torso", "Stack" .. sx, V3(sx * 0.62, 9.9, 1.5))
		A.vents[#A.vents + 1] = att("Torso", "VentS" .. sx, V3(sx * 2.4, 8.2, 0.3))
		detW("Torso", "Block", V3(0.12, 2.6, 0.12), CF(sx * 0.9, 9.2, 1.9), BLACK)    -- antenas
	end
	gear("Torso", "GearBackA", V3(-0.85, 7.95, 2.0), "z", 0.85, 0.25, 0.9)
	gear("Torso", "GearBackB", V3(0.7, 7.55, 2.0), "z", 0.6, 0.25, -1.3)
	-- esterno + painel de LEDs (como o desenho) + marcas vermelhas
	detW("Torso", "Block", V3(0.55, 2.5, 0.25), CF(0, 7.0, -0.9), BONE, SMOOTH)
	detW("Torso", "Block", V3(0.95, 0.95, 0.12), CF(0, 7.55, -1.05), STEEL_D, METAL)
	for i = -1, 1 do
		A.leds[#A.leds + 1] = detW("Torso", "Ball", V3(0.13, 0.13, 0.13), CF(i * 0.28, 7.85, -1.13), i == -1 and REDC or Color3.fromRGB(40, 40, 40), NEON)
	end
	A.oval = detW("Torso", "Cyl", V3(0.06, 0.5, 0.34), CF(0, 7.3, -1.13) * cylZ, REDC, NEON)
	for i = 0, 1 do
		A.slashes[#A.slashes + 1] = detW("Torso", "Block", V3(0.7, 0.08, 0.06), CF(0.75, 8.05 - i * 0.18, -0.95) * AN(0, 0, -0.6), REDC, NEON)
	end
	A.core = detW("Torso", "Cyl", V3(1.6, 0.42, 0.42), CF(0, 7.0, 0.2) * cylY, Color3.fromRGB(110, 14, 10), NEON)
	-- costelas (abrem para engolir o operador)
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		node("Rib" .. s, "Torso", "Block", V3(0.14, 2.5, 0.14), CF(sx * 0.95, 7.0, -0.1), V3(sx * 0.15, 7.0, -0.85), BONE, SMOOTH)
		for i = 0, 4 do
			local y = 5.95 + i * 0.5
			bar("Rib" .. s, V3(sx * 0.15, y, -0.85), V3(sx * 0.95, y, -0.1), 0.13, BONE, SMOOTH)
			bar("Rib" .. s, V3(sx * 0.95, y, -0.1), V3(sx * 0.35, y, 0.45), 0.13, BONE, SMOOTH)
		end
		detW("Rib" .. s, "Block", V3(0.35, 0.5, 0.35), CF(sx * 0.4, 7.0, 0.45), BLACK, METAL)
		A.leds[#A.leds + 1] = detW("Rib" .. s, "Ball", V3(0.1, 0.1, 0.1), CF(sx * 0.4, 7.2, 0.28), REDC, NEON)
	end

	-- CABEÇA (crânio) + olhos + mandíbula -----------------------------------------
	node("Head", "Torso", "Block", V3(1.5, 1.6, 1.5), CF(0, 9.75, -0.1), V3(0, 8.6, 0.1), BONE, SMOOTH)
	for _, sx in ipairs({ -1, 1 }) do
		detW("Head", "Block", V3(0.2, 1.0, 0.9), CF(sx * 0.8, 9.5, -0.2), BONE_D, SMOOTH)
		detW("Head", "Cyl", V3(0.3, 0.66, 0.66), CF(sx * 0.38, 9.9, -0.78) * cylZ, BLACK, SMOOTH)
		detW("Head", "Cyl", V3(0.2, 0.7, 0.7), CF(sx * 0.86, 9.8, 0.1), STEEL)
		detW("Head", "Ball", V3(0.12, 0.12, 0.12), CF(sx * 0.8, 10.2, -0.3), REDC, NEON)
	end
	detW("Head", "Block", V3(0.1, 0.4, 0.1), CF(0, 9.55, -0.78), BLACK, SMOOTH)
	detW("Head", "Block", V3(1.5, 0.25, 0.3), CF(0, 10.3, -0.7), BONE_D, SMOOTH)
	detW("Head", "Block", V3(1.2, 1.0, 0.3), CF(0, 9.8, 0.7), STEEL)
	A.headLED = detW("Head", "Ball", V3(0.24, 0.24, 0.24), CF(0.3, 10.58, -0.3), REDC, NEON)
	for i = 0, 5 do
		local a = (i / 6) * PI * 2
		bar("Head", V3(0.55 * math.cos(a), 10.5, 0.55 * math.sin(a) - 0.1), V3(0.85 * math.cos(a), 11.15, 0.85 * math.sin(a) - 0.1), 0.07, BLACK, METAL)
	end
	for _, sx in ipairs({ -1, 1 }) do
		local e = node("Eye" .. (sx < 0 and "L" or "R"), "Head", "Cyl", V3(0.08, 0.36, 0.36), CF(sx * 0.38, 9.9, -0.93) * cylZ, V3(sx * 0.38, 9.9, -0.93), WHITE, NEON)
		detW(e.name, "Cyl", V3(0.1, 0.12, 0.12), CF(sx * 0.38, 9.9, -0.97) * cylZ, BLACK, SMOOTH)
		A.pupils[#A.pupils + 1] = e
	end
	node("Jaw", "Head", "Block", V3(1.2, 0.4, 1.1), CF(0, 8.85, -0.2), V3(0, 9.05, 0.35), BONE_D, SMOOTH)
	for i = 0, 5 do detW("Jaw", "Block", V3(0.12, 0.2, 0.1), CF(-0.5 + i * 0.2, 9.0, -0.72), BONE, SMOOTH) end
	detW("Jaw", "Block", V3(0.8, 0.12, 0.3), CF(0, 8.62, -0.65), BLACK, METAL)
	A.eyeAtt = att("Head", "EyeLight", V3(0, 9.9, -1.1))

	-- BRAÇOS (esquerdo = -X) -----------------------------------------------------
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		local Sh, E, W = V3(sx * 2.1, 8.1, 0), V3(sx * 2.1, 5.9, 0), V3(sx * 2.1, 3.7, 0)
		local u, f, h = "UpperArm" .. s, "Forearm" .. s, "Hand" .. s
		bnode(u, "Torso", Sh, E, 0.5, BONE, SMOOTH)
		local pad = detW(u, "Block", V3(1.5, 0.5, 1.6), CF(sx * 2.1, 8.4, 0) * AN(0, 0, -sx * 0.12), OLIVE)
		detW(u, "Ball", V3(0.75, 0.75, 0.75), CF(Sh), BLACK, METAL)
		detW(u, "Block", V3(0.65, 0.22, 0.65), CF(sx * 2.1, 7.0, 0), BLACK, SMOOTH)
		detW(u, "Block", V3(0.65, 0.22, 0.65), CF(sx * 2.1, 6.4, 0), BLACK, SMOOTH)
		if sx < 0 then
			local plate = detW(u, "Block", V3(1.0, 0.06, 0.6), CF(sx * 2.1, 8.66, 0), STEEL, METAL)
			local sg = Instance.new("SurfaceGui")
			sg.Face = Enum.NormalId.Top; sg.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud; sg.PixelsPerStud = 60; sg.Parent = plate
			local tl = label(sg, Enum.Font.GothamBlack, Color3.fromRGB(20, 22, 18))
			tl.Size = UDim2.fromScale(1, 1); tl.Text = "0.X1"
		end
		gear(u, "GearSh" .. s, V3(sx * 2.75, 7.95, 0), "x", 0.75, 0.25, 1.0 * sx)
		bnode(f, u, E, W, 0.45, BONE, SMOOTH)
		detW(f, "Ball", V3(0.65, 0.65, 0.65), CF(E), BLACK, METAL)
		detW(f, "Block", V3(0.7, 1.1, 0.2), CF(sx * 2.1, 4.8, -0.4), OLIVE)
		for i = 0, 2 do detW(f, "Block", V3(0.62, 0.2, 0.62), CF(sx * 2.1, 5.5 - i * 0.7, 0), BLACK, SMOOTH) end
		node(h, f, "Block", V3(0.8, 0.5, 0.8), CF(sx * 2.1, 3.45, 0), W, BLACK, METAL)
		detW(h, "Block", V3(0.95, 0.35, 0.95), CF(sx * 2.1, 3.7, 0), BLACK, SMOOTH)
		for i = 1, 3 do
			local x = sx * 2.1 + (i - 2) * 0.3
			local ja, jb, tip = V3(x, 3.2, -0.05), V3(x, 1.9, -0.05), V3(x, 0.8, -0.05)
			bnode("FA" .. s .. i, h, ja, jb, 0.17, BONE, SMOOTH)
			bnode("FB" .. s .. i, "FA" .. s .. i, jb, tip, 0.12, BONE, SMOOTH)
			detW("FB" .. s .. i, "Block", V3(0.1, 0.1, 0.35), CF(x, 0.7, -0.05) * AN(0.4, 0, 0), BLACK, METAL)
		end
		-- broca (aparece ao escavar): 3 cilindros decrescentes
		for k, d in ipairs({ 0.95, 0.68, 0.4 }) do
			local dr = detW(h, "Cyl", V3(0.6, d, d), CF(sx * 2.1, 3.1 - (k - 1) * 0.55, 0) * cylY, k == 2 and STEEL or STEEL_D, METAL)
			A.drill[#A.drill + 1] = dr
		end
		A.pads = A.pads or {}
		A.pads[#A.pads + 1] = pad
	end

	-- PERNAS ----------------------------------------------------------------------
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		local H, K, An = V3(sx * HIPX, 4.6, 0), V3(sx * HIPX, 2.6, 0), V3(sx * HIPX, ANKLE_H, 0)
		local th, sh, ft = "Thigh" .. s, "Shin" .. s, "Foot" .. s
		bnode(th, "Pelvis", H, K, 0.45, BONE, SMOOTH, true)
		detW(th, "Ball", V3(0.7, 0.7, 0.7), CF(H), BLACK, METAL)
		detW(th, "Block", V3(0.9, 1.2, 0.25), CF(sx * HIPX, 3.9, -0.45), OLIVE)
		detW(th, "Block", V3(0.7, 0.2, 0.7), CF(sx * HIPX, 3.3, 0), BLACK, SMOOTH)
		detW(th, "Block", V3(0.7, 0.2, 0.7), CF(sx * HIPX, 4.1, 0), BLACK, SMOOTH)
		detW(th, "Ball", V3(0.65, 0.65, 0.65), CF(K), BLACK, METAL)
		gear(th, "GearKnee" .. s, V3(sx * 1.78, 2.6, 0), "x", 0.55, 0.2, -1.1 * sx)
		bnode(sh, th, K, An, 0.4, BONE, SMOOTH, true)
		detW(sh, "Block", V3(0.7, 1.1, 0.2), CF(sx * HIPX, 1.8, 0.4), STEEL)
		detW(sh, "Block", V3(0.6, 0.2, 0.6), CF(sx * HIPX, 1.2, 0), BLACK, SMOOTH)
		detW(sh, "Cyl", V3(0.3, 0.7, 0.7), CF(An) * cylY, BLACK, METAL)
		node(ft, sh, "Block", V3(1.3, 0.45, 2.2), CF(sx * HIPX, 0.225, -0.45), An, STEEL_D, METAL, true)
		for i = 0, 2 do detW(ft, "Block", V3(0.15, 0.15, 0.7), CF(sx * HIPX + (i - 1) * 0.4, 0.15, -1.8), BLACK, METAL) end
		detW(ft, "Block", V3(1.2, 0.5, 0.5), CF(sx * HIPX, 0.5, 0.55), OLIVE)
		detW(ft, "Block", V3(1.4, 0.12, 2.3), CF(sx * HIPX, 0.06, -0.45), BLACK, DIAMOND)
		A["fp" .. s] = att(ft, "FootPt" .. s, V3(sx * HIPX, 0.05, -0.45))
		A.vents[#A.vents + 1] = att(th, "VentK" .. s, V3(sx * 1.6, 2.7, 0.2))
	end

	-- desancora (menos a pelve) e cria a junta de pilotagem com o HumanoidRootPart
	local pelvis = N.Pelvis.part
	for _, d in ipairs(model:GetDescendants()) do
		if d:IsA("BasePart") and d ~= pelvis then d.Anchored = false end
	end
	model.PrimaryPart = pelvis
	local link = Instance.new("Motor6D"); link.Name = "PilotLink"; link.Part1 = pelvis; link.Parent = pelvis
	A.link = link
	A.solids = { A.hull, N.FootL.part, N.FootR.part }
	for _, p in ipairs(A.solids) do p.CanCollide = true end

	-- luzes
	local function light(parent, color, range)
		local l = Instance.new("PointLight"); l.Color = color; l.Range = range; l.Brightness = 0; l.Parent = parent
		return l
	end
	A.eyeLight = light(A.eyeAtt, REDC, 18)
	A.coreLight = light(N.Torso.part, REDC, 14)
	-- sons
	for key, def in pairs(CFG.Sounds) do
		if def.Id and def.Id ~= "" then
			local snd = Instance.new("Sound")
			snd.Name = "Prypot_" .. key; snd.SoundId = def.Id; snd.Volume = def.Volume or 1
			snd.RollOffMode = Enum.RollOffMode.InverseTapered; snd.RollOffMaxDistance = 220
			snd.Parent = N.Torso.part
			A.sounds[key] = snd
		end
	end

	-- vapor nas juntas / fumaça nas chaminés
	A.steam, A.smoke = {}, {}
	for _, a in ipairs(A.vents) do local e = FXm.steam(); e.Parent = a; A.steam[#A.steam + 1] = e end
	for _, a in ipairs(A.stacks) do local e = FXm.smoke(); e.Parent = a; A.smoke[#A.smoke + 1] = e end
	A.sparkAtt = Instance.new("Attachment"); A.sparkAtt.Parent = N.Torso.part
	A.spark = FXm.spark(); A.spark.Parent = A.sparkAtt
	A.dustAtt = Instance.new("Attachment"); A.dustAtt.Parent = pelvis
	A.dust = FXm.dust(); A.dust.Parent = A.dustAtt

	-- etiqueta 3D
	local bb = Instance.new("BillboardGui")
	bb.Name = "PrypotTag"; bb.Adornee = N.Head.part; bb.Size = UDim2.fromOffset(200, 50)
	bb.StudsOffset = V3(0, 3, 0); bb.AlwaysOnTop = true; bb.MaxDistance = 140
	bb.Parent = player:WaitForChild("PlayerGui")
	local t1 = label(bb, Enum.Font.GothamBlack, CFG.Colors.Accent)
	t1.Size = UDim2.fromScale(1, 0.55); t1.Text = CFG.Name; t1.TextStrokeTransparency = 0.4
	local t2 = label(bb, Enum.Font.GothamMedium, Color3.new(1, 1, 1))
	t2.Position = UDim2.fromScale(0, 0.55); t2.Size = UDim2.fromScale(1, 0.38); t2.TextStrokeTransparency = 0.4
	A.tag, A.tagText = bb, t2

	-- ===== estruturas dinâmicas (atualizadas a cada frame por posição no mundo) =====
	A.dynFolder = Instance.new("Folder"); A.dynFolder.Name = "PrypotDyn"; A.dynFolder.Parent = Workspace
	local function dyn(shape, size, color, mat)
		local p = Instance.new("Part")
		p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false; p.CastShadow = false
		p.Size = size; p.Color = color; p.Material = mat or METAL; p.TopSurface = Enum.SurfaceType.Smooth
		if shape == "Cyl" then p.Shape = Enum.PartType.Cylinder end
		p.Parent = A.dynFolder
		parts[#parts + 1] = p
		return p
	end
	A.dyn = dyn
	local function piston(an, aw, bn, bw, r)
		A.pistons[#A.pistons + 1] = { a = anchorW(an, aw), b = anchorW(bn, bw), outer = dyn("Cyl", V3(1, r * 2, r * 2), STEEL), inner = dyn("Cyl", V3(1, r * 1.2, r * 1.2), BRASS) }
	end
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		piston("Thigh" .. s, V3(sx * HIPX, 3.9, -0.62), "Shin" .. s, V3(sx * HIPX, 1.7, -0.52), 0.1)
		piston("UpperArm" .. s, V3(sx * 2.1, 6.7, 0.42), "Forearm" .. s, V3(sx * 2.1, 5.2, 0.42), 0.08)
		piston("Pelvis", V3(sx * 1.9, 4.85, -0.45), "Thigh" .. s, V3(sx * HIPX, 4.0, -0.35), 0.09)
		-- mola (coil em zigue-zague) do coxa-trás até o calcanhar: amortece pousos e pulos
		local sp = { a = anchorW("Thigh" .. s, V3(sx * HIPX, 3.7, 0.5)), b = anchorW("Foot" .. s, V3(sx * HIPX, 0.75, 0.55)), segs = {}, n = DETAIL > 1 and 10 or 7 }
		for i = 1, sp.n do sp.segs[i] = dyn("Cyl", V3(1, 0.07, 0.07), STEEL) end
		sp.capA, sp.capB = dyn("Cyl", V3(0.12, 0.34, 0.34), BLACK), dyn("Cyl", V3(0.12, 0.34, 0.34), BLACK)
		A.springs[#A.springs + 1] = sp
	end
	piston("Torso", V3(0, 8.3, -0.3), "Head", V3(0, 9.2, -0.5), 0.07)

	-- cabos com física Verlet (fios da cabeça como no desenho, mangueiras, cabos pendurados)
	local CN = Q().cables
	local function cable(an, aw, bn, bw, length, r, color)
		local c = { a = anchorW(an, aw), b = bn and anchorW(bn, bw) or nil, n = CN, L = length / (CN - 1), pts = {}, prev = {}, seg = {}, r = r }
		for i = 1, CN do c.seg[i] = i < CN and dyn("Cyl", V3(1, r * 2, r * 2), color) or nil end
		A.cables[#A.cables + 1] = c
	end
	cable("Head", V3(0.3, 10.5, -0.3), "Torso", V3(0.9, 8.4, 0.6), 2.8, 0.045, BLACK)
	cable("Head", V3(0.4, 10.5, -0.2), "Torso", V3(1.1, 8.3, 0.4), 3.1, 0.04, STEEL_D)
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		cable("Torso", V3(sx * 0.7, 8.2, 1.6), "UpperArm" .. s, V3(sx * 2.1, 6.3, 0.25), 4.6, 0.06, BLACK)
		cable("Torso", V3(sx * 0.6, 6.0, 1.2), "Thigh" .. s, V3(sx * HIPX, 3.9, 0.4), 4.4, 0.06, BLACK)
		cable("Pelvis", V3(sx * 0.9, 4.6, 0.55), nil, nil, 3.2, 0.05, STEEL_D)
	end

	-- braços auxiliares (aranha): IK de 2 ossos desenhado em peças soltas
	for _, sx in ipairs({ -1, 1 }) do
		for k = 1, 2 do
			local lm = { root = anchorW("Torso", V3(sx * (0.9 - (k - 1) * 0.1), 8.0 - (k - 1) * 1.4, 1.0)), side = sx, k = k, l1 = 2.7, l2 = 2.5,
				tip = V3(0, 0, 0), seg = { dyn("Cyl", V3(1, 0.14, 0.14), BONE, SMOOTH), dyn("Cyl", V3(1, 0.11, 0.11), BONE, SMOOTH) },
				joints = { dyn("Ball", V3(0.26, 0.26, 0.26), BLACK), dyn("Ball", V3(0.22, 0.22, 0.22), BLACK) }, claws = {} }
			lm.joints[1].Shape = Enum.PartType.Ball; lm.joints[2].Shape = Enum.PartType.Ball
			for i = 1, 3 do lm.claws[i] = dyn("Block", V3(0.07, 0.07, 0.6), BLACK) end
			A.aux[#A.aux + 1] = lm
		end
	end

	-- agulhas neurais (5 tubos com ponta) — só aparecem na cutscene de entrada/saída
	for i = 1, 5 do
		local pr = { root = anchorW("Torso", V3(-0.4 + (i - 1) * 0.2, 6.55 + ((i - 1) % 2) * 0.2 - 0.1, 1.3)), seg = {}, tip = dyn("Block", V3(0.07, 0.07, 0.8), STEEL), n = 7 }
		for j = 1, pr.n - 1 do pr.seg[j] = dyn("Cyl", V3(1, 0.09, 0.09), BLACK) end
		pr.tip.Transparency = 1
		for _, sgm in ipairs(pr.seg) do sgm.Transparency = 1 end
		A.probes[#A.probes + 1] = pr
	end
	-- rastro neural na coluna e linhas até o crânio (luz que sobe)
	for _, d in ipairs(A.drill) do d.LocalTransparencyModifier = 1 end
	return A
end
