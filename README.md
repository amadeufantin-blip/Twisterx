local WindUI = loadstring(game:HttpGet("https://github.com/Footagesus/WindUI/releases/latest/download/main.lua"))()

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game.Workspace

local LP = Players.LocalPlayer
local gv = getgenv()

local prevTeardown = gv and gv.TWISTEXX_teardown
if type(prevTeardown) == "function" then
	pcall(prevTeardown)
end

local alive = true
local connections = {}

local function connect(signal, fn)
	local con = signal:Connect(fn)
	table.insert(connections, con)
	return con
end

local cfg = {
	god = false,
	farm = false,
	peak = 0.1,
	esp = false,
	anti = false,
	night = false,
	bring = true,
	fly = false,
	orbit = false,
	fspeed = 1,
	autocar = false,
	sonda = false,
	sondaCam = false,
}
if gv then gv.TWISTEXX_cfg = cfg end

local Window = WindUI:CreateWindow({
	Title = "TWISTERX HUB",
	Author = "TwisterX Hub",
	Folder = "TWISTEXX",
	Theme = "Dark",
	ToggleKey = Enum.KeyCode.Insert,
	Size = UDim2.fromOffset(520, 460),
})

local FarmTab = Window:Tab({ Title = "Farm", Icon = "settings" })
local CarTab = Window:Tab({ Title = "Carro", Icon = "sliders" })
local EspTab = Window:Tab({ Title = "ESP", Icon = "layout" })
local SondaTab = Window:Tab({ Title = "Sonda", Icon = "bird" })

local MoneyHolder = LP:WaitForChild("leaderstats", 10)
local Money = MoneyHolder and MoneyHolder:FindFirstChild("Money")

local storms = {}
local lastStormRefresh = 0

local ANTI_NAMES = { "Speedlimit", "NormalSpeed", "DirtSpeed", "GrassSpeed" }

local function refreshStorms()
	lastStormRefresh = os.clock()
	local list = {}
	local direct = Workspace:FindFirstChild("Storm")
	if direct then
		local car = direct:FindFirstChild("Car1")
		if car then table.insert(list, car) end
	end
	local folder = Workspace:FindFirstChild("Storms")
	if folder then
		for _, storm in ipairs(folder:GetChildren()) do
			for _, car in ipairs(storm:GetChildren()) do
				if car.Name == "Car1" then table.insert(list, car) end
			end
		end
	end
	local keep = {}
	for _, car in ipairs(list) do
		if car.Parent ~= nil then table.insert(keep, car) end
	end
	storms = keep
end

local stormIgnoreCache = {}
local lastStormIgnoreRefresh = 0

local STORM_MODEL_MATCH = function(name)
	local n = string.lower(name)
	return n:find("storm", 1, false) ~= nil or n:find("tornado", 1, false) ~= nil
end

local STORM_PART_MATCH = function(name)
	local n = string.lower(name)
	return n:match("splash") ~= nil
		or n:match("cloud") ~= nil
		or n:match("rain") ~= nil
		or n:match("meso") ~= nil
		or n:match("funnel") ~= nil
		or n:match("wallcloud") ~= nil
		or n:match("storm") ~= nil
		or n:match("tornado") ~= nil
		or n:match("particle") ~= nil
end

local function refreshStormIgnore()
	lastStormIgnoreRefresh = os.clock()
	local seen = {}
	local result = {}
	local function addPart(part)
		if part and part:IsA("BasePart") and not seen[part] then
			seen[part] = true
			result[#result + 1] = part
		end
	end
	local function addModel(model)
		if not model or not model:IsA("Model") then return end
		for _, d in ipairs(model:GetDescendants()) do
			addPart(d)
		end
	end
	for _, rootChild in ipairs(Workspace:GetChildren()) do
		if rootChild:IsA("Model") and STORM_MODEL_MATCH(rootChild.Name) then
			addModel(rootChild)
		elseif rootChild:IsA("BasePart") and STORM_PART_MATCH(rootChild.Name) then
			addPart(rootChild)
		end
	end
	stormIgnoreCache = result
end

local function allStormIgnore()
	if os.clock() - lastStormIgnoreRefresh >= 1 then refreshStormIgnore() end
	return stormIgnoreCache
end

local SKY_Y = 6000

local function groundYFromSky(x, z, ignoreExtra)
	local ignore = {}
	for _, d in ipairs(allStormIgnore()) do ignore[#ignore + 1] = d end
	if ignoreExtra then
		for _, d in ipairs(ignoreExtra) do ignore[#ignore + 1] = d end
	end
	local rayp = RaycastParams.new()
	rayp.FilterType = Enum.RaycastFilterType.Exclude
	rayp.FilterDescendantsInstances = ignore
	for attempt = 1, 16 do
		local res = Workspace:Raycast(Vector3.new(x, SKY_Y, z), Vector3.new(0, -(SKY_Y + 1600), 0), rayp)
		if not res then return nil end
		local hit = res.Instance
		local hitName = hit.Name or ""
		if not hit:IsA("BasePart") or STORM_PART_MATCH(hitName) or hit.Anchored == false then
			ignore[#ignore + 1] = hit
			rayp.FilterDescendantsInstances = ignore
		else
			local gy = res.Position.Y
			if gy > SKY_Y - 200 or gy < -800 then
				ignore[#ignore + 1] = hit
				rayp.FilterDescendantsInstances = ignore
			else
				return gy
			end
		end
	end
	local fallback = Workspace:Raycast(Vector3.new(x, SKY_Y, z), Vector3.new(0, -(SKY_Y + 1600), 0), rayp)
	return fallback and fallback.Position.Y or nil
end

local function stormWind(car)
	local w = car:FindFirstChild("Windspeed")
	if w and w:IsA("ValueBase") then
		return w.Value + 20
	end
	return car:GetAttribute("WindSpeeds") or 0
end

local function stormRadius(car)
	local mesh = car:FindFirstChildOfClass("SpecialMesh")
	if mesh and mesh.Scale then
		return mesh.Scale.X
	end
	return car:GetAttribute("OuterRadius") or 100
end

local function bestStorm()
	if #storms == 0 or os.clock() - lastStormRefresh >= 1 then refreshStorms() end
	if #storms == 0 then return nil end
	local best, bestW = nil, -1
	for _, car in ipairs(storms) do
		if car.Parent ~= nil then
			local wv = stormWind(car)
			if wv > bestW then
				best, bestW = car, wv
			end
		end
	end
	if not best then return nil end
	local radius = stormRadius(best)
	return best, radius, bestW, best.Position
end

local function windAt(center, radius, wind, pos)
	local dx = pos.X - center.X
	local dz = pos.Z - center.Z
	local mag = math.sqrt(dx * dx + dz * dz)
	if mag > radius then return 0 end
	if mag < radius * 0.1 then
		return 10 + (mag / (radius * 0.1)) * (wind * 0.9 - 10)
	end
	return wind * (1 - mag / radius)
end

FarmTab:Toggle({
	Title = "God Mode",
	Desc = "Vida sempre no maximo",
	Value = false,
	Callback = function(v)
		cfg.god = v
	end,
})

FarmTab:Space()

FarmTab:Toggle({
	Title = "Visao Noturna",
	Desc = "Clareia a noite (brilho, luz ambiente e neblina curta) para conseguir enxergar no server publico",
	Value = false,
	Callback = function(v)
		cfg.night = v
		if v then
			WindUI:Notify({ Title = "TWISTERX", Content = "Visao noturna ligada", Duration = 2, Icon = "moon" })
		end
	end,
})

FarmTab:Space()

FarmTab:Toggle({
	Title = "Farm Automatico",
	Desc = "Segue o ponto de vento maximo do tornado (de pe ou dirigindo o carro)",
	Value = false,
	Callback = function(v)
		cfg.farm = v
	end,
})

FarmTab:Space()

FarmTab:Slider({
	Title = "Distancia do nucleo",
	Desc = "Fracao do raio pro melhor vento (padrao 10%)",
	Value = { Min = 0.05, Max = 0.4, Default = 0.1 },
	Step = 0.01,
	Callback = function(v)
		cfg.peak = v
	end,
})

FarmTab:Space()

FarmTab:Toggle({
	Title = "Orbita no fim do farm",
	Desc = "Em vez de parar no ponto de vento maximo, o carro circula o nucleo a 35% do raio coletando vento continuamente",
	Value = false,
	Callback = function(v)
		cfg.orbit = v
	end,
})

FarmTab:Space()

FarmTab:Slider({
	Title = "Velocidade do farm",
	Desc = "Quao rapido o carro desliza ate o ponto de vento maximo (1 = suave, 3 = agressivo). Teletransporte so acontece se estiver MUITO longe do tornado.",
	Value = { Min = 0.5, Max = 3, Default = 1 },
	Step = 0.1,
	Callback = function(v)
		cfg.fspeed = v
	end,
})

FarmTab:Space()

FarmTab:Button({
	Title = "Ir para o tornado",
	Color = Color3.fromHex("#10C550"),
	Justify = "Center",
	Icon = "",
	Callback = function()
		local car, radius, _, center = bestStorm()
		if not (car and radius > 1 and center) then
			WindUI:Notify({ Title = "TWISTERX", Content = "Nenhum tornado ativo no momento", Duration = 4, Icon = "bell" })
			return
		end
		local char = LP.Character
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		if not hrp then
			WindUI:Notify({ Title = "TWISTERX", Content = "Personagem nao encontrado", Duration = 4, Icon = "bell" })
			return
		end
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if hum and hum.SeatPart and hum.SeatPart:IsA("VehicleSeat") then
			hum.Sit = false
			task.wait(0.1)
		end
		local dx = hrp.Position.X - center.X
		local dz = hrp.Position.Z - center.Z
		local m = math.sqrt(dx * dx + dz * dz)
		if m < 1 then dx, dz, m = 1, 0, 1 end
		local off = radius * cfg.peak
		local tx = center.X + dx / m * off
		local tz = center.Z + dz / m * off
		local gy = groundYFromSky(tx, tz, { char })
		hrp.CFrame = CFrame.new(tx, (gy or center.Y) + 4, tz)
		hrp.AssemblyLinearVelocity = Vector3.zero
		WindUI:Notify({ Title = "TWISTERX", Content = "Teleportado pro tornado", Duration = 3, Icon = "bird" })
	end,
})

FarmTab:Space()

FarmTab:Toggle({
	Title = "Voo do tornado (Peter)",
	Desc = "Cena do Into the Storm: te joga pra dentro do funil girando no ar. Se estiver dirigindo, o carro gira junto (camera primeira pessoa). Desligue pra pousar.",
	Value = false,
	Callback = function(v)
		cfg.fly = v
		if v then
			cfg.farm = false
		else
			local char = LP.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			local seat = hum and hum.Sit and hum.SeatPart or nil
			if hum and not (seat and seat:IsA("VehicleSeat")) then
				hum.PlatformStand = false
			end
			local carModel = (seat and seat:IsA("VehicleSeat") and seat.Parent) or nil
			if carModel and carModel:IsA("Model") then
				local cp = carModel:GetPivot()
				local gy = groundYFromSky(cp.Position.X, cp.Position.Z, { carModel, char })
				if gy then
					local rot = carModel:GetPivot() - carModel:GetPivot().Position
					carModel:PivotTo(CFrame.new(cp.Position.X, gy + 1, cp.Position.Z) * rot)
					for _, d in carModel:GetDescendants() do
						if d:IsA("BasePart") then
							d.AssemblyLinearVelocity = Vector3.zero
							d.AssemblyAngularVelocity = Vector3.zero
						end
					end
				end
			elseif hum and hum:FindFirstChild("HumanoidRootPart") then
				local hrp = char:FindFirstChild("HumanoidRootPart")
				local gy = groundYFromSky(hrp.Position.X, hrp.Position.Z, { char })
				if gy then
					local cf = hrp.CFrame
					hrp.CFrame = CFrame.new(cf.X, gy + 4, cf.Z) * (cf - cf.Position)
				end
			end
		end
	end,
})

FarmTab:Space()

local StatusPara = FarmTab:Paragraph({
	Title = "Status",
	Desc = "aguardando...",
})

FarmTab:Paragraph({
	Title = "Como ganha dinheiro",
	Desc = "O jogo paga quando voce bate seu recorde pessoal de vento (SaveWindSpeed). O vento maximo fica a ~10% do raio do nucleo do tornado. Farm automatico te segura la.",
	Color = Color3.fromHex("#83889E"),
})

FarmTab:Space()

local carName = "Dominator 1"
local carList = {
	{ Display = "Dominator 1 [gratis]", Name = "Dominator 1" },
	{ Display = "Dominator 1 2009 [gratis]", Name = "Dominator 1 2009" },
	{ Display = "Dominator 2 [gratis]", Name = "Dominator 2" },
	{ Display = "Dominator 3 [gratis]", Name = "Dominator 3" },
	{ Display = "Dominator 3 no rockets [gratis]", Name = "Dominator 3 no rockets" },
	{ Display = "Dominator Fore [gratis]", Name = "Dominator Fore" },
	{ Display = "TIV 1 2009 [gratis]", Name = "TIV 1 2009" },
	{ Display = "TIV 1 2003 [gratis]", Name = "TIV 1 2003" },
	{ Display = "TIV 2 [gratis]", Name = "TIV 2" },
	{ Display = "armadillo [gratis]", Name = "armadillo" },
	{ Display = "Dorothy [gratis]", Name = "Dorothy" },
	{ Display = "ole blue [gratis]", Name = "ole blue" },
	{ Display = "Doghouse 2009 [gratis]", Name = "Doghouse 2009" },
	{ Display = "Doghouse 2012 [gratis]", Name = "Doghouse 2012" },
	{ Display = "Vortex 2 Scout [gratis]", Name = "Vortex 2 Scout" },
	{ Display = "Vortex 2 Mobile Mesonet [gratis]", Name = "Vortex 2 Mobile Mesonet" },
	{ Display = "TP [gratis]", Name = "TP" },
	{ Display = "The Titan [gratis]", Name = "The Titan" },
	{ Display = "Titus [gratis]", Name = "Titus" },
	{ Display = "Tornado Attack [gratis]", Name = "Tornado Attack" },
	{ Display = "UTAV [gratis]", Name = "UTAV" },
	{ Display = "Support Vehicle Of Dorothy [gratis]", Name = "Support Vehicle Of Dorothy" },
	{ Display = "Suba-TIV [$9]", Name = "Suba-TIV" },
	{ Display = "Tornado Wrangler [$1k]", Name = "Tornado Wrangler" },
	{ Display = "hash tv van [$2k]", Name = "hash tv van" },
	{ Display = "HashTVChasingTeam [$2k]", Name = "HashTVChasingTeam" },
	{ Display = "HashTVMobile [$2k]", Name = "HashTVMobile" },
	{ Display = "Tornado Car [$50k]", Name = "Tornado Car" },
	{ Display = "Storm Piercer [$50k]", Name = "Storm Piercer" },
	{ Display = "TIV 2 2009 [$100k]", Name = "TIV 2 2009" },
	{ Display = "DOW 3 [$200k]", Name = "DOW 3" },
	{ Display = "raxpol [$325k]", Name = "raxpol" },
	{ Display = "DOW 6 [$500k]", Name = "DOW 6" },
	{ Display = "Lancer1 [$1M]", Name = "Lancer1" },
	{ Display = "Lancer2015 [$10M]", Name = "Lancer2015" },
	{ Display = "MDS [precisa badge]", Name = "MDS" },
}

local carDisplayValues = {}
local carNameByDisplay = {}
for _, e in ipairs(carList) do
	table.insert(carDisplayValues, e.Display)
	carNameByDisplay[e.Display] = e.Name
end

CarTab:Dropdown({
	Title = "Lista de carros",
	Desc = "Clique no carro. Todos saem de graca (o servidor nao valida saldo)",
	Values = carDisplayValues,
	Value = "Dominator 1 [gratis]",
	Callback = function(selected)
		if selected and carNameByDisplay[selected] then
			carName = carNameByDisplay[selected]
			WindUI:Notify({ Title = "TWISTERX", Content = "Carro: " .. carName, Duration = 2, Icon = "car" })
		end
	end,
})

CarTab:Space()

local manualCarInput = CarTab:Input({
	Title = "Nome manual (opcional)",
	Value = "Dominator 1",
	Placeholder = "digite o nome exato e use Spawnar",
	Callback = function(v)
		carName = v
	end,
})

CarTab:Space()

local remoteCache = {}

local function findRemote(name)
	local cached = remoteCache[name]
	if cached and cached.Parent then return cached end
	local found = nil
	local function walk(root, depth)
		if found or not root or depth > 6 then return end
		for _, child in ipairs(root:GetChildren()) do
			if (child:IsA("RemoteEvent") or child:IsA("RemoteFunction")) and child.Name == name then
				found = child
				return
			end
			walk(child, depth + 1)
		end
	end
	walk(game.ReplicatedStorage, 0)
	walk(game:GetService("ReplicatedFirst"), 0)
	walk(Workspace, 0)
	if found then remoteCache[name] = found end
	return found
end

local function fireRemote(name, ...)
	local args = table.pack(...)
	local r = findRemote(name)
	if not r then
		WindUI:Notify({ Title = "TWISTERX", Content = "Remote '" .. name .. "' nao encontrado", Duration = 4, Icon = "bell" })
		return false
	end
	if r:IsA("RemoteFunction") then
		task.spawn(function()
			pcall(function()
				r:InvokeServer(table.unpack(args))
			end)
		end)
	else
		pcall(function()
			r:FireServer(table.unpack(args))
		end)
	end
	return true
end

local PAYMENT = {
	["Suba-TIV"] = -9,
	["Tornado Wrangler"] = -1000,
	["hash tv van"] = -2000,
	["HashTVChasingTeam"] = -2000,
	["HashTVMobile"] = -2000,
	["Tornado Car"] = 0,
	["Storm Piercer"] = 0,
	["TIV 2 2009"] = "noarg",
	["DOW 3"] = "noarg",
	["DOW 6"] = "noarg",
	["raxpol"] = "noarg",
	["Lancer1"] = 0,
	["Lancer2015"] = 0,
	["MDS"] = 0,
}

local function getOwnCar()
	return Workspace:FindFirstChild(LP.Name .. "sCar")
end

local function findStoreRemote(carName)
	local store = LP.PlayerGui:FindFirstChild("Store")
	if not store then return nil end
	local found = nil
	local function walk(node)
		if found or not node then return end
		if node.Name == carName then
			local re = node:FindFirstChild("RemoteEvent")
			if re then found = re end
			return
		end
		for _, c in ipairs(node:GetChildren()) do
			walk(c)
		end
	end
	walk(store)
	return found
end

local function spawnCar()
	local radar = game.ReplicatedStorage:FindFirstChild("OnRadar")
	if radar and radar.Value == true then
		WindUI:Notify({ Title = "TWISTERX", Content = "Dentro do radar (missao) nao da pra spawnar carro", Duration = 4, Icon = "bell" })
		return
	end
	local char = LP.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	if humanoid and humanoid.SeatPart and humanoid.SeatPart:IsA("VehicleSeat") then
		humanoid.Sit = false
		task.wait(0.15)
	end
	local oldCar = getOwnCar()
	if oldCar then
		pcall(function()
			fireRemote("DeleteCar", oldCar)
		end)
		task.wait(0.25)
	end
	if not fireRemote("SpawnCar", carName) then return end
	local pay = PAYMENT[carName]
	if pay then
		local re = findStoreRemote(carName)
		if re then
			pcall(function()
				re:FireServer(0)
			end)
		end
	end
	if cfg.bring then
		local wantPivot = char and char:GetPivot()
		task.spawn(function()
			local waited = 0
			while alive and waited < 4 do
				task.wait(0.2)
				waited += 0.2
				local car = getOwnCar()
				if car then
					local cf = car:GetPivot()
					local base = wantPivot or cf
					local dir = base.LookVector
					local target = base.Position + Vector3.new(dir.X, 0, dir.Z) * 12
					target = Vector3.new(target.X, cf.Position.Y, target.Z)
					car:PivotTo((cf - cf.Position) + target)
					break
				end
			end
		end)
	end
	WindUI:Notify({ Title = "TWISTERX", Content = "Spawnado '" .. carName .. "'", Duration = 3, Icon = "bird" })
end

CarTab:Button({
	Title = "Spawnar carro",
	Color = Color3.fromHex("#10C550"),
	Justify = "Center",
	Icon = "",
	Callback = function()
		spawnCar()
	end,
})

CarTab:Space()

CarTab:Button({
	Title = "Remover carro",
	Color = Color3.fromHex("#EF4F1D"),
	Justify = "Center",
	Icon = "",
	Callback = function()
		local car = getOwnCar()
		if not car then
			WindUI:Notify({ Title = "TWISTERX", Content = "Nenhum " .. LP.Name .. "sCar no mapa", Duration = 3, Icon = "trash" })
			return
		end
		local humanoid = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
		if humanoid and humanoid.SeatPart and humanoid.SeatPart:IsA("VehicleSeat") then
			humanoid.Sit = false
			task.wait(0.15)
		end
		if fireRemote("DeleteCar", car) then
			WindUI:Notify({ Title = "TWISTERX", Content = "Removendo seu carro", Duration = 3, Icon = "trash" })
		end
	end,
})

CarTab:Space()

CarTab:Toggle({
	Title = "Trazer carro ate voce",
	Desc = "Teleporta o carro recem spawnado pra frente do personagem",
	Value = true,
	Callback = function(v)
		cfg.bring = v
	end,
})

CarTab:Toggle({
	Title = "Auto-carro no farm",
	Desc = "Com o farm ligado, se nao tiver carro spawna um novo e senta no banco automaticamente (uma vez por sessao)",
	Value = false,
	Callback = function(v)
		cfg.autocar = v
	end,
})

CarTab:Toggle({
	Title = "Anti-Bloqueio",
	Desc = "Remove o bloqueio de dirigir quando o carro planta (cap de velocidade no maximo + desenterra se afundar)",
	Value = false,
	Callback = function(v)
		cfg.anti = v
	end,
})

CarTab:Space()

CarTab:Paragraph({
	Title = "Dica",
	Desc = "Liga o farm dirigindo o carro: ele te leva pro ponto de vento maximo usando PivotTo (sem ancorar). Se o carro cair no void, ele volta sozinho. Anti-Bloqueio segue deixando o carro dirigir sem travar.",
	Color = Color3.fromHex("#83889E"),
})

EspTab:Toggle({
	Title = "ESP do tornado",
	Desc = "Circulo vermelho = raio real, verde = ponto de farm; tracer + placa do funil e dos outros jogadores",
	Value = false,
	Callback = function(v)
		cfg.esp = v
	end,
})

local SondaStatus = nil

local sondaModel: Model? = nil
local sondaBody: BasePart? = nil
local sondaLight: BasePart? = nil
local sondaBlink = false
local sondaPrevCam: Enum.CameraType? = nil
local sondaPrevSubject: Instance? = nil

local function removeSonda()
	if sondaLight then
		pcall(function()
			sondaLight:Destroy()
		end)
	end
	if sondaModel then
		pcall(function()
			sondaModel:Destroy()
		end)
	end
	sondaModel = nil
	sondaBody = nil
	sondaLight = nil
	if SondaStatus then
		SondaStatus:SetDesc("nenhuma sonda")
	end
end

local function detachSondaCamera()
	pcall(function()
		local cam = Workspace.CurrentCamera
		if sondaPrevCam then
			cam.CameraType = sondaPrevCam
		else
			cam.CameraType = Enum.CameraType.Custom
		end
		if sondaPrevSubject then
			cam.CameraSubject = sondaPrevSubject
		end
	end)
	sondaPrevCam = nil
	sondaPrevSubject = nil
end

local function attachSondaCamera()
	pcall(function()
		local cam = Workspace.CurrentCamera
		sondaPrevCam = cam.CameraType
		local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
		sondaPrevSubject = hum or cam.CameraSubject
	end)
end

local function spawnSonda()
	removeSonda()
	local char = LP.Character
	local origin = char and char:GetPivot().Position or Vector3.zero
	origin = origin + Vector3.new(0, 4, -12)
	local model = Instance.new("Model")
	model.Name = "TWISTERX_Sonda"
	model.Parent = Workspace
	local body = Instance.new("Part")
	body.Name = "Corpo"
	body.Size = Vector3.new(1.2, 1.2, 1.2)
	body.Shape = Enum.PartType.Ball
	body.Material = Enum.Material.Metal
	body.Color = Color3.fromRGB(220, 60, 40)
	body.Anchored = true
	body.CanCollide = false
	body.Massless = true
	body.CastShadow = false
	body.Parent = model
	local spike = Instance.new("Part")
	spike.Name = "Antena"
	spike.Size = Vector3.new(0.15, 1.4, 0.15)
	spike.Material = Enum.Material.Metal
	spike.Color = Color3.fromRGB(40, 40, 45)
	spike.Anchored = true
	spike.CanCollide = false
	spike.Massless = true
	spike.CastShadow = false
	spike.Parent = model
	local light = Instance.new("Part")
	light.Name = "Led"
	light.Size = Vector3.new(0.3, 0.3, 0.3)
	light.Shape = Enum.PartType.Ball
	light.Material = Enum.Material.Neon
	light.Color = Color3.fromRGB(255, 60, 40)
	light.Anchored = true
	light.CanCollide = false
	light.Massless = true
	light.CastShadow = false
	light.Parent = model
	local pt = Instance.new("PointLight")
	pt.Color = Color3.fromRGB(255, 80, 50)
	pt.Brightness = 3
	pt.Range = 12
	pt.Parent = light
	model:PivotTo(CFrame.new(origin))
	sondaModel = model
	sondaBody = body
	sondaLight = light
	SondaStatus:SetDesc("sonda lancada, indo pro funil")
end

SondaTab:Button({
	Title = "Lancar sonda",
	Color = Color3.fromHex("#10C550"),
	Justify = "Center",
	Icon = "",
	Callback = function()
		cfg.sonda = true
		spawnSonda()
	end,
})

SondaTab:Space()

SondaTab:Toggle({
	Title = "Camera na sonda",
	Desc = "Gruda a camera na sonda girando dentro do funil",
	Value = false,
	Callback = function(v)
		cfg.sondaCam = v
		if v then
			attachSondaCamera()
		else
			detachSondaCamera()
		end
	end,
})

SondaTab:Space()

SondaTab:Button({
	Title = "Remover sonda",
	Color = Color3.fromHex("#EF4F1D"),
	Justify = "Center",
	Icon = "",
	Callback = function()
		cfg.sonda = false
		cfg.sondaCam = false
		detachSondaCamera()
		removeSonda()
	end,
})

SondaTab:Space()

local SondaStatusParagraph = SondaTab:Paragraph({
	Title = "Status",
	Desc = "nenhuma sonda",
})
SondaStatus = SondaStatusParagraph

local lines = {}
local circles = {}
local texts = {}

local function getLine(index)
	local i = index + 1
	while #lines < i do
		table.insert(lines, Drawing.new("Line"))
	end
	return lines[i]
end

local function getCircle(index)
	local i = index + 1
	while #circles < i do
		table.insert(circles, Drawing.new("Circle"))
	end
	return circles[i]
end

local function getText(index)
	local i = index + 1
	while #texts < i do
		table.insert(texts, Drawing.new("Text"))
	end
	return texts[i]
end

local function drawSquare(cam, center, half, color, base)
	local p1, on1 = cam:WorldToViewportPoint(center + Vector3.new(-half, 0, -half))
	local p2, on2 = cam:WorldToViewportPoint(center + Vector3.new(half, 0, -half))
	local p3, on3 = cam:WorldToViewportPoint(center + Vector3.new(half, 0, half))
	local p4, on4 = cam:WorldToViewportPoint(center + Vector3.new(-half, 0, half))
	local visible = on1 and on2 and on3 and on4
	base.Visible = visible
	if not visible then return end
	base.From = Vector2.new(p1.X, p1.Y)
	base.To = Vector2.new(p2.X, p2.Y)
	base.Color = color
	base.Thickness = 2
end

local cam = Workspace.CurrentCamera
local playerCache = {}

local function scanPlayers()
	playerCache = {}
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= LP then
			local root = plr.Character and plr.Character:FindFirstChild("HumanoidRootPart")
			if root then
				table.insert(playerCache, { name = plr.Name, root = root })
			end
		end
	end
end

connect(RunService.RenderStepped, function()
	if not cfg.esp then
		for _, l in ipairs(lines) do l.Visible = false end
		for _, c in ipairs(circles) do c.Visible = false end
		for _, t in ipairs(texts) do t.Visible = false end
		return
	end
	local car, radius, _, center = bestStorm()
	if car and center and radius > 1 then
		local ring = getCircle(1)
		ring.Visible = true
		ring.Position = cam:WorldToViewportPoint(center + Vector3.new(0, 1, 0))
		ring.Radius = radius
		ring.Thickness = 2
		ring.Color = Color3.fromRGB(255, 60, 60)
		ring.Filled = false
		local farmPoint = center + (Vector3.new(center.X, 0, center.Z) - Vector3.new(center.X, 0, center.Z)) * 0
		local fp = cam:WorldToViewportPoint(center + Vector3.new(radius * cfg.peak, 1, 0))
		local fpGreen = getCircle(2)
		fpGreen.Visible = true
		fpGreen.Position = Vector2.new(fp.X, fp.Y)
		fpGreen.Radius = 6
		fpGreen.Thickness = 2
		fpGreen.Color = Color3.fromRGB(80, 255, 120)
		fpGreen.Filled = false
		if cfg.farm then
			local char = LP.Character
			local hrp = char and char:FindFirstChild("HumanoidRootPart")
			if hrp then
				local sea = char:FindFirstChildOfClass("Humanoid")
				local cm = nil
				local ss = sea and sea.Sit and sea.SeatPart or nil
				if ss and ss:IsA("VehicleSeat") then cm = ss.Parent end
				local basePos = (cm and cm:IsA("Model") and cm:GetPivot().Position) or hrp.Position
				local dx = basePos.X - center.X
				local dz = basePos.Z - center.Z
				local mm = math.sqrt(dx * dx + dz * dz)
				if mm < 1 then dx, dz, mm = 1, 0, 1 end
				local off = radius * cfg.peak
				local tx = center.X + dx / mm * off
				local tz = center.Z + dz / mm * off
				local tp, onT = cam:WorldToViewportPoint(Vector3.new(tx, basePos.Y + 2, tz))
				local bp, onB = cam:WorldToViewportPoint(basePos + Vector3.new(0, 2, 0))
				if onT and onB then
					local fl = getLine(50)
					fl.Visible = true
					fl.From = Vector2.new(bp.X, bp.Y)
					fl.To = Vector2.new(tp.X, tp.Y)
					fl.Color = Color3.fromRGB(80, 255, 120)
					fl.Thickness = 1
					local ft = getText(99)
					ft.Visible = true
					ft.Position = Vector2.new(tp.X, tp.Y - 10)
					ft.Text = string.format("FARM %.0fm", (Vector3.new(basePos.X, 0, basePos.Z) - Vector3.new(tx, 0, tz)).Magnitude)
					ft.Size = 12
					ft.Color = Color3.fromRGB(80, 255, 120)
					ft.Outline = true
					ft.Center = true
					ft.Font = Drawing.Fonts.UI
				end
			end
		end
	end
	scanPlayers()
	local camPos = cam.CFrame.Position
	local camLook = cam.CFrame.LookVector
	local idx = 1
	for _, q in ipairs(playerCache) do
		local r = q.root
		if r and r.Parent then
			local toP = r.Position - camPos
			local dist = toP.Magnitude
			if dist > 1 then
				toP = toP / dist
				local facing = camLook:Dot(toP)
				if facing > 0.1 then
					local sp, onS = cam:WorldToViewportPoint(r.Position + Vector3.new(0, 3, 0))
					if onS then
						local line = getLine(100 + idx)
						line.Visible = true
						line.From = Vector2.new(sp.X, sp.Y)
						line.To = Vector2.new(sp.X, sp.Y + 30)
						line.Color = Color3.fromRGB(120, 200, 255)
						line.Thickness = 2
						local nxt = getText(idx)
						nxt.Visible = true
						nxt.Position = Vector2.new(sp.X, sp.Y - 18)
						nxt.Text = string.format("%s [%dm]", q.name, math.floor(dist))
						nxt.Size = 13
						nxt.Color = Color3.fromRGB(120, 200, 255)
						nxt.Outline = true
						nxt.Center = true
						nxt.Font = Drawing.Fonts.UI
					end
					idx = idx + 1
				end
			end
		end
	end
end)

local lastStatusAt = 0
local lastMoney = Money and Money.Value or 0
local lastRateT = 0
local rate = 0
local noStormNotified = 0
local lastAntiTick = 0
local lastFarmMove = 0
local farmWasOn = false
local anchoredCar = nil
local anchorLift = 0
local autoCarDone = false

task.spawn(function()
	while alive do
		task.wait()
		if not cfg.sonda or not sondaModel then continue end
		local storm, radius, _, center = bestStorm()
		if not (storm and radius > 1 and center) then continue end
		local t = os.clock()
		local body = sondaBody
		if not body or body.Parent == nil then
			cfg.sonda = false
			continue
		end
		local pos = body.Position
		local dx = pos.X - center.X
		local dz = pos.Z - center.Z
		local m = math.sqrt(dx * dx + dz * dz)
		local targetR = radius * 0.25
		if m > targetR * 1.5 then
			local speed = math.clamp(m * 0.08, 4, 60)
			local nx = pos.X - dx / m * speed
			local nz = pos.Z - dz / m * speed
			local ny = pos.Y + (center.Y + 20 - pos.Y) * 0.06
			sondaModel:PivotTo(CFrame.new(nx, ny, nz) * CFrame.Angles(0, t * 2, 0))
		else
			local ang = t * 1.4
			local ox = math.cos(ang) * targetR
			local oz = math.sin(ang) * targetR
			local oy = center.Y + 25 + math.sin(t * 2.2) * 8
			sondaModel:PivotTo(CFrame.new(center.X + ox, oy, center.Z + oz) * CFrame.Angles(math.sin(t * 3) * 0.4, ang, math.cos(t * 3) * 0.4))
		end
		sondaBlink = not sondaBlink
		if sondaLight then
			sondaLight.Color = sondaBlink and Color3.fromRGB(255, 120, 60) or Color3.fromRGB(255, 40, 20)
		end
		if cfg.sondaCam then
			pcall(function()
				local cam = Workspace.CurrentCamera
				cam.CameraType = Enum.CameraType.Scriptable
				cam.CFrame = CFrame.new(body.Position, center + Vector3.new(0, 40, 0))
			end)
		end
	end
end)

connect(RunService.Heartbeat, function()
	local char = LP.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local hrp = char and char:FindFirstChild("HumanoidRootPart")

	if cfg.night then
		local l = game:GetService("Lighting")
		l.Brightness = 2
		l.Ambient = Color3.fromRGB(100, 100, 100)
		l.ClockTime = 14
		l.GlobalShadows = false
		l.FogEnd = 5000
		l.FogStart = 500
	end

	local function getPivotPart(model)
		local p = model.PrimaryPart
		if p and p:IsA("BasePart") then
			return p
		end
		local best, bestM
		for _, d in model:GetDescendants() do
			if d:IsA("BasePart") then
				local s = d.Size
				local v = s.X * s.Y * s.Z
				if not bestM or v > bestM then
					best, bestM = d, v
				end
			end
		end
		if best then
			model.PrimaryPart = best
		end
		return best
	end

	local function isWheelPart(part)
		if not part or not part:IsA("BasePart") then return false end
		local n = string.lower(part.Name)
		if n:find("wheel", 1, true) or n:find("tire", 1, true) or n:find("tyre", 1, true) or n:find("rim", 1, true) then
			return true
		end
		for _, c in ipairs(part:GetChildren()) do
			if c:IsA("CylinderConstraint") or c:IsA("HingeConstraint") then
				return true
			end
		end
		return false
	end

	local function snapModel(model, targetPos)
		local parts = model:GetDescendants()
		local ref = model.PrimaryPart
		if not ref or not ref:IsA("BasePart") then
			for _, d in ipairs(parts) do
				if d:IsA("BasePart") then
					ref = d
					break
				end
			end
		end
		if not ref then
			return false
		end
		local delta = targetPos - ref.Position
		local lowest = nil
		for _, d in ipairs(parts) do
			if d:IsA("BasePart") and not isWheelPart(d) then
				d.CFrame = d.CFrame + delta
				d.AssemblyLinearVelocity = Vector3.zero
				d.AssemblyAngularVelocity = Vector3.zero
				local y = d.Position.Y - d.Size.Y / 2
				if not lowest or y < lowest then lowest = y end
			end
		end
		return true, lowest or ref.Position.Y
	end

	local function setCarAnchored(model, val)
		if not model or not model:IsA("Model") then return end
		local drive = nil
		for _, d in model:GetDescendants() do
			if d:IsA("BasePart") then
				if isWheelPart(d) then
					d.Anchored = false
				else
					d.Anchored = val
				end
			elseif not drive and d:IsA("LocalScript") and d.Name == "Drive" then
				drive = d
			end
		end
		if drive then
			drive.Disabled = val
		end
	end

	local function carLowestY(model, refPart)
		local ref = refPart or model.PrimaryPart
		local lowest = nil
		for _, d in model:GetDescendants() do
			if d:IsA("BasePart") then
				if ref and math.abs(d.Position.Y - ref.Position.Y) > 200 then
					continue
				end
				local y = d.Position.Y - d.Size.Y / 2
				if not lowest or y < lowest then lowest = y end
			end
		end
		return lowest or 0
	end

	if cfg.farm ~= farmWasOn then
		if anchoredCar then
			setCarAnchored(anchoredCar, false)
			anchoredCar = nil
			anchorLift = 0
		end
		farmWasOn = cfg.farm
		if not cfg.farm then
			autoCarDone = false
		end
	end

	if cfg.farm and hrp and hum then
		local seat = hum.Sit and hum.SeatPart or nil
		local carModel = (seat and seat:IsA("VehicleSeat") and seat.Parent) or nil
		if carModel and not carModel:IsA("Model") then carModel = nil end

		if not carModel and cfg.autocar and not autoCarDone then
			autoCarDone = true
			local own = getOwnCar()
			if not own or not own:IsA("Model") then
				spawnCar()
			end
		end

		if not carModel and cfg.autocar and autoCarDone then
			local own = getOwnCar()
			if own and own:IsA("Model") then
				local vehicleSeat = nil
				for _, d in own:GetDescendants() do
					if d:IsA("VehicleSeat") then
						vehicleSeat = d
						break
					end
				end
				if vehicleSeat then
					hrp.CFrame = vehicleSeat.CFrame + Vector3.new(0, 0, 0)
					task.wait(0.1)
					hum.Sit = true
				end
			end
		end

		local fnow = os.clock()
		if fnow - lastFarmMove >= 0.05 then
			lastFarmMove = fnow
			local storm, radius, wind, center = bestStorm()
			if storm and radius > 1 and center then
				local base = hrp.Position
				if carModel then base = carModel:GetPivot().Position end
				local dx = base.X - center.X
				local dz = base.Z - center.Z
				local m = math.sqrt(dx * dx + dz * dz)
				if m < 1 then
					dx, dz, m = 1, 0, 1
				end
				local off = radius * cfg.peak
				local tx, tz
				if cfg.orbit then
					local orbitR = math.clamp(radius * 0.35, 15, 120)
					local ang = os.clock() * 1.1
					tx = center.X + math.cos(ang) * orbitR
					tz = center.Z + math.sin(ang) * orbitR
				else
					tx = center.X + dx / m * off
					tz = center.Z + dz / m * off
				end

				local ignore = { char }
				if carModel then ignore[#ignore + 1] = carModel end
				local groundY = groundYFromSky(tx, tz, ignore)
				local curWind = windAt(center, radius, wind, base)
				local far = (Vector3.new(base.X, 0, base.Z) - Vector3.new(tx, 0, tz)).Magnitude
				if groundY and (far > (cfg.orbit and 8 or 3) or curWind < wind * 0.8) then
					if carModel then
						local pivot = getPivotPart(carModel)
						if pivot then
							local low = carLowestY(carModel, pivot)
							local lift = math.clamp(pivot.Position.Y - low, 0, 8)
							local rot = carModel:GetPivot() - carModel:GetPivot().Position
							local targetPos = Vector3.new(tx, groundY + lift, tz)
							local fromPos = pivot.Position
							local dist = (fromPos - targetPos).Magnitude
							if dist > 300 then
								carModel:PivotTo(CFrame.new(targetPos) * rot)
							else
								local speed = cfg.fspeed
								local step = math.clamp(dist * 0.06 * speed, 0.5, dist)
								local newPos = fromPos:Lerp(targetPos, step / math.max(dist, 1))
								carModel:PivotTo(CFrame.new(newPos) * rot)
							end
							for _, d in carModel:GetDescendants() do
								if d:IsA("BasePart") then
									d.AssemblyLinearVelocity = Vector3.zero
									d.AssemblyAngularVelocity = Vector3.zero
								end
							end
						end
					else
						hrp.CFrame = CFrame.new(tx, groundY + 4, tz)
						hrp.AssemblyLinearVelocity = Vector3.zero
					end
				end
			end
		end
	end

	if cfg.anti and hum then
		local seat = hum.Sit and hum.SeatPart or nil
		if seat then
			if seat:IsA("VehicleSeat") and seat.MaxSpeed < 5000 then seat.MaxSpeed = 5000 end
			local carModel = seat.Parent
			if carModel and carModel:IsA("Model") then
				for _, name in ipairs(ANTI_NAMES) do
					local val = carModel:FindFirstChild(name)
					if val and val:IsA("NumberValue") and val.Value ~= 999 then
						val.Value = 999
					end
				end
				local prim = getPivotPart(carModel)
				if prim and os.clock() - lastAntiTick > 0.4 then
					lastAntiTick = os.clock()
					local pos = prim.Position
					if prim.AssemblyLinearVelocity.Magnitude < 4 and prim.AssemblyAngularVelocity.Magnitude < 1 then
						local ground = groundYFromSky(pos.X, pos.Z, { carModel, char })
						if ground and ground < pos.Y + 15 and pos.Y < ground - 0.5 then
							snapModel(carModel, Vector3.new(pos.X, ground + 2, pos.Z))
						end
					end
				end
			end
		end
	end

	local now = os.clock()
	if now - lastStatusAt < 0.5 then return end
	lastStatusAt = now

	local m = Money and Money.Value or 0
	if m ~= lastMoney then
		local dt = now - lastRateT
		lastRateT = now
		if dt > 0 then
			local inst = (m - lastMoney) / dt
			rate = rate * 0.6 + inst * 0.4
		end
		lastMoney = m
	elseif now - lastRateT > 8 then
		rate = rate * 0.5
	end

	local status = string.format("Dinheiro: $%.0f   |   ~$%.0f/min", m, rate)
	if hrp then
		local car, radius, wind, center = bestStorm()
		if car and radius > 1 then
			local w = windAt(center, radius, wind, hrp.Position)
			status = status .. string.format("   |   Vento: %.0f mph   |   Tornado: %.0f mph", w, wind)
		elseif cfg.farm and now - noStormNotified > 15 then
			noStormNotified = now
			WindUI:Notify({ Title = "TWISTERX", Content = "Nenhum tornado ativo no momento", Duration = 4, Icon = "bell" })
		end
	end
	StatusPara:SetDesc(status)
end)

task.spawn(function()
	while alive do
		if cfg.god then
			local char = LP.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if hum then
				if hum.Health < hum.MaxHealth then hum.Health = hum.MaxHealth end
			end
		end
		task.wait()
	end
end)

task.spawn(function()
	while alive do
		task.wait()
		if not cfg.fly then continue end
		local char = LP.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		if not (hrp and hum) then continue end
		local seat = hum.Sit and hum.SeatPart or nil
		local carModel = (seat and seat:IsA("VehicleSeat") and seat.Parent) or nil
		if carModel and not carModel:IsA("Model") then carModel = nil end
		local storm, radius, _, center = bestStorm()
		if storm and radius > 1 and center then
			local t = os.clock() * 4
			local orbitR = math.clamp(radius * 0.35, 15, 120)
			local spinY = os.clock() * 9
			local x = center.X + math.cos(t) * orbitR
			local z = center.Z + math.sin(t) * orbitR
			local y = center.Y + 30 + math.sin(os.clock() * 2.5) * 18
			if carModel then
				local carPivot = carModel:GetPivot()
				local rotTarget = CFrame.Angles(math.sin(spinY) * 1.6, spinY, math.cos(spinY) * 1.1)
				local cf = CFrame.new(x, y, z) * rotTarget
				local fwd = cf.LookVector
				local camCF = CFrame.new(cf.Position, cf.Position + fwd)
				carModel:PivotTo(camCF)
				for _, d in carModel:GetDescendants() do
					if d:IsA("BasePart") then
						d.AssemblyLinearVelocity = Vector3.zero
						d.AssemblyAngularVelocity = Vector3.zero
					end
				end
			else
				local cf = CFrame.new(x, y, z) * CFrame.Angles(math.sin(spinY) * 1.6, spinY, math.cos(spinY) * 1.1)
				hrp.CFrame = cf
				hrp.AssemblyLinearVelocity = Vector3.zero
				hrp.AssemblyAngularVelocity = Vector3.zero
				hum.PlatformStand = true
				pcall(function()
					hrp.CustomPhysicalProperties = PhysicalProperties.new(0, 0, 0, 0, 0)
				end)
			end
			if not cfg.god then
				hum.Health = hum.MaxHealth
			end
		end
	end
end)

task.spawn(function()
	while alive do
		task.wait(0.5)
		if not cfg.fly then continue end
		local char = LP.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not hum then continue end
		if hum.Sit and hum.SeatPart then continue end
		if hum.PlatformStand == false then
			hum.PlatformStand = true
		end
	end
end)

task.spawn(function()
	while alive do
		task.wait(0.5)
		local car = workspace:FindFirstChild(LP.Name .. "sCar")
		if not (car and car:IsA("Model")) then continue end
		if not (cfg.farm or cfg.anti or cfg.fly) then continue end
		local low = nil
		for _, d in car:GetDescendants() do
			if d:IsA("BasePart") then
				local y = d.Position.Y - d.Size.Y / 2
				if not low or y < low then low = y end
			end
		end
		if low and low < -500 then
			local char = LP.Character
			local hrp = char and char:FindFirstChild("HumanoidRootPart")
			local target = hrp and hrp.Position or car:GetPivot().Position
			local ground = groundYFromSky(target.X, target.Z, { car, char })
			local rot = car:GetPivot() - car:GetPivot().Position
			car:PivotTo(CFrame.new(target.X, (ground or target.Y) + 3, target.Z) * rot)
			WindUI:Notify({ Title = "TWISTERX", Content = "Carro caiu no void, puxado de volta", Duration = 3, Icon = "alert" })
		end
	end
end)

local function teardown()
	if not alive then return end
	alive = false
	for _, con in ipairs(connections) do
		pcall(function()
			con:Disconnect()
		end)
	end
	connections = {}
	for _, l in ipairs(lines) do
		pcall(function()
			l:Remove()
		end)
	end
	lines = {}
	for _, c in ipairs(circles) do
		pcall(function()
			c:Remove()
		end)
	end
	circles = {}
	for _, t in ipairs(texts) do
		pcall(function()
			t:Remove()
		end)
	end
	texts = {}
	detachSondaCamera()
	removeSonda()
	pcall(function()
		Window:Destroy()
	end)
	if gv then
		gv.TWISTEXX_teardown = nil
		gv.TWISTEXX_cfg = nil
	end
end

if gv then gv.TWISTEXX_teardown = teardown end
