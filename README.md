-- [[ 
--     Script: Central de Troll
--     Marca: RISE SHOP👑
--     Tema: Verde Esmeralda / Neon
-- ]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local TextChatService = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualUser = game:GetService("VirtualUser")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = workspace.CurrentCamera

-- Carrega Kavo com proteção
local okKavo, Kavo = pcall(function()
	return loadstring(game:HttpGet("https://raw.githubusercontent.com/xHeptc/Kavo-UI-Library/main/source.lua"))()
end)

if not okKavo or not Kavo then
	warn("RISE SHOP👑: Falha ao carregar Kavo UI.")
	return
end

local GreenTheme = {
	SchemeColor = Color3.fromRGB(16, 185, 129),
	Background = Color3.fromRGB(10, 22, 16),
	Header = Color3.fromRGB(12, 32, 22),
	TextColor = Color3.fromRGB(236, 253, 245),
	ElementColor = Color3.fromRGB(16, 42, 28)
}

local Window = Kavo.CreateLib("RISE SHOP👑 • PAINEL TROLL", GreenTheme)

-- // Estados //
local SelectedTargetName = "Nenhum"
local SelectedTarget = nil

local TouchFlingEnabled = false
local FlingPower = 45000
local TouchPushEnabled = false
local PushPower = 250

local OrbitEnabled = false
local OrbitRadius = 5
local OrbitSpeed = 14
local OrbitAngle = 0

local AttachMode = "Nenhum" -- "Nenhum", "Cabeca", "Costas", "Frente", "Embaixo"
local FollowLoopEnabled = false
local SandwichEnabled = false
local SandwichSide = 1
local SurroundBlocksEnabled = false
local CageEnabled = false
local SurroundParts = {}
local CageParts = {}

local FlyEnabled = false
local FlySpeed = 70
local NoClipEnabled = false
local SuperJumpEnabled = false
local SuperJumpPower = 120
local PlatformStandEnabled = false
local ReverseMoveEnabled = false
local AntiAFKEnabled = true

local SpamJumpEnabled = false
local SpamSitEnabled = false
local SpamDanceLoopEnabled = false
local SitEverywhereEnabled = false
local SpamToolEquipEnabled = false
local SpamToolActivateEnabled = false
local FreezePosEnabled = false
local FrozenCFrame = nil
local SpinEnabled = false
local SpinSpeed = 50

local AutoCycleTargetEnabled = false
local lastCycleTime = 0

local AutoRejoinEnabled = true
local ConfigFileName = "RiseShopTrollConfig.json"
local ChatSpamEnabled = false
local ChatSpamText = "RISE SHOP👑"
local ChatSpamDelay = 3
local lastChatTime = 0
local touchCooldowns = {}

-- // Helpers //
local function Notify(title, text, duration)
	pcall(function()
		game:GetService("StarterGui"):SetCore("SendNotification", {
			Title = title or "RISE SHOP👑",
			Text = text or "",
			Duration = duration or 3
		})
	end)
end

local function GetPlayerList()
	local list = { "Nenhum" }
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer then
			table.insert(list, p.Name)
		end
	end
	return list
end

local function ClearPartsTable(tbl)
	for _, part in ipairs(tbl) do
		if part and part.Parent then
			part:Destroy()
		end
	end
	table.clear(tbl)
end

local function ClearTargetStructures()
	ClearPartsTable(SurroundParts)
	ClearPartsTable(CageParts)
end

local function RefreshTarget()
	if SelectedTargetName == "" or SelectedTargetName == "Nenhum" then
		SelectedTarget = nil
		ClearTargetStructures()
		return
	end
	SelectedTarget = Players:FindFirstChild(SelectedTargetName)
end

local function GetValidTarget()
	RefreshTarget()
	if SelectedTarget and SelectedTarget.Character then
		local hrp = SelectedTarget.Character:FindFirstChild("HumanoidRootPart")
		local hum = SelectedTarget.Character:FindFirstChildOfClass("Humanoid")
		if hrp and hum and hum.Health > 0 then
			return SelectedTarget, SelectedTarget.Character, hrp, hum
		end
	end
	return nil, nil, nil, nil
end

local function GetLocalChar()
	local char = LocalPlayer.Character
	if not char then return nil, nil, nil end
	return char, char:FindFirstChild("HumanoidRootPart"), char:FindFirstChildOfClass("Humanoid")
end

-- // Remover Alvo e Voltar ao Normal //
local function ResetTargetAndNormal()
	SelectedTargetName = "Nenhum"
	SelectedTarget = nil

	OrbitEnabled = false
	AttachMode = "Nenhum"
	FollowLoopEnabled = false
	SandwichEnabled = false
	SurroundBlocksEnabled = false
	CageEnabled = false
	AutoCycleTargetEnabled = false
	ClearTargetStructures()

	local char, hrp, hum = GetLocalChar()
	if hum then
		hum.PlatformStand = false
		hum.Sit = false
	end
	if hrp then
		hrp.Anchored = false
		hrp.AssemblyLinearVelocity = Vector3.zero
		hrp.AssemblyAngularVelocity = Vector3.zero
		hrp.Velocity = Vector3.zero
		hrp.RotVelocity = Vector3.zero
	end
	if char then
		for _, part in ipairs(char:GetDescendants()) do
			if part:IsA("BasePart") then
				part.CanCollide = true
			end
		end
	end

	Notify("RISE SHOP👑", "Alvo removido e tudo voltou ao normal.", 3)
end

local function StopAllTrolls()
	TouchFlingEnabled = false
	TouchPushEnabled = false
	FlyEnabled = false
	NoClipEnabled = false
	SuperJumpEnabled = false
	PlatformStandEnabled = false
	ReverseMoveEnabled = false
	SpamJumpEnabled = false
	SpamSitEnabled = false
	SpamDanceLoopEnabled = false
	SitEverywhereEnabled = false
	SpamToolEquipEnabled = false
	SpamToolActivateEnabled = false
	FreezePosEnabled = false
	FrozenCFrame = nil
	SpinEnabled = false
	ChatSpamEnabled = false
	ResetTargetAndNormal()
end

-- // Fling Real por Colisão Física //
local function FlingPlayer(targetPlayer)
	if not targetPlayer or targetPlayer == LocalPlayer then return end
	local tChar = targetPlayer.Character
	local tHrp = tChar and tChar:FindFirstChild("HumanoidRootPart")
	local myChar, myHrp, myHum = GetLocalChar()
	if not tHrp or not myHrp or not myHum then return end

	local savedCF = myHrp.CFrame
	local oldCollide = {}
	for _, part in ipairs(myChar:GetDescendants()) do
		if part:IsA("BasePart") then
			oldCollide[part] = part.CanCollide
			part.CanCollide = true
		end
	end

	myHum.PlatformStand = true
	local bav = Instance.new("BodyAngularVelocity")
	bav.Name = "RiseFlingSpin"
	bav.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
	bav.AngularVelocity = Vector3.new(0, FlingPower, 0)
	bav.P = math.huge
	bav.Parent = myHrp

	local start = tick()
	while tick() - start < 0.45 and tHrp and tHrp.Parent do
		myHrp.CFrame = tHrp.CFrame
		myHrp.AssemblyLinearVelocity = Vector3.new(FlingPower * 0.25, FlingPower * 0.25, FlingPower * 0.25)
		myHrp.AssemblyAngularVelocity = Vector3.new(0, FlingPower, 0)
		RunService.Heartbeat:Wait()
	end

	if bav and bav.Parent then bav:Destroy() end
	myHum.PlatformStand = false
	myHrp.AssemblyLinearVelocity = Vector3.zero
	myHrp.AssemblyAngularVelocity = Vector3.zero
	myHrp.CFrame = savedCF

	for part, state in pairs(oldCollide) do
		if part and part.Parent then
			part.CanCollide = state
		end
	end
end

local function PushPlayer(targetPlayer)
	if not targetPlayer or targetPlayer == LocalPlayer then return end
	local tChar = targetPlayer.Character
	local tHrp = tChar and tChar:FindFirstChild("HumanoidRootPart")
	local _, myHrp = GetLocalChar()
	if not tHrp or not myHrp then return end

	local dir = (tHrp.Position - myHrp.Position)
	if dir.Magnitude < 0.1 then
		dir = Vector3.new(1, 0.4, 0)
	else
		dir = (dir.Unit + Vector3.new(0, 0.35, 0)).Unit
	end

	pcall(function()
		tHrp.AssemblyLinearVelocity = dir * PushPower
		tHrp.Velocity = dir * PushPower
	end)
end

local function OnTouchPlayer(hitPlayer)
	if not hitPlayer or hitPlayer == LocalPlayer then return end
	local now = tick()
	if touchCooldowns[hitPlayer] and now - touchCooldowns[hitPlayer] < 0.2 then
		return
	end
	touchCooldowns[hitPlayer] = now

	if TouchFlingEnabled then
		task.spawn(function()
			FlingPlayer(hitPlayer)
		end)
	elseif TouchPushEnabled then
		PushPlayer(hitPlayer)
	end
end

local function BindCharacterTouch(char)
	if not char then return end
	for _, part in ipairs(char:GetDescendants()) do
		if part:IsA("BasePart") then
			part.Touched:Connect(function(hit)
				if not TouchFlingEnabled and not TouchPushEnabled then return end
				local model = hit:FindFirstAncestorOfClass("Model")
				if model then
					local plr = Players:GetPlayerFromCharacter(model)
					if plr and plr ~= LocalPlayer then
						OnTouchPlayer(plr)
					end
				end
			end)
		end
	end
end

-- // Blocos / Jaula ao Redor do Alvo //
local function UpdateTargetStructures(targetHrp)
	if SurroundBlocksEnabled and targetHrp then
		if #SurroundParts == 0 then
			for i = 1, 6 do
				local p = Instance.new("Part")
				p.Name = "RiseSurroundBlock"
				p.Size = Vector3.new(2, 4, 2)
				p.Anchored = true
				p.CanCollide = true
				p.Transparency = 0.35
				p.Material = Enum.Material.Neon
				p.Color = Color3.fromRGB(16, 185, 129)
				p.Parent = workspace
				table.insert(SurroundParts, p)
			end
		end
		for i, p in ipairs(SurroundParts) do
			local angle = OrbitAngle + (i * (math.pi * 2 / #SurroundParts))
			local pos = targetHrp.Position + Vector3.new(math.cos(angle) * 4.5, 0, math.sin(angle) * 4.5)
			p.CFrame = CFrame.new(pos, targetHrp.Position)
		end
	elseif #SurroundParts > 0 then
		ClearPartsTable(SurroundParts)
	end

	if CageEnabled and targetHrp then
		if #CageParts == 0 then
			local offsets = {
				Vector3.new(3, 0, 0),
				Vector3.new(-3, 0, 0),
				Vector3.new(0, 0, 3),
				Vector3.new(0, 0, -3),
				Vector3.new(0, 4, 0)
			}
			for idx, off in ipairs(offsets) do
				local p = Instance.new("Part")
				p.Name = "RiseCagePart"
				p.Size = (idx == 5) and Vector3.new(7, 1, 7) or Vector3.new(1, 6, 7)
				p.Anchored = true
				p.CanCollide = true
				p.Transparency = 0.4
				p.Material = Enum.Material.Neon
				p.Color = Color3.fromRGB(16, 185, 129)
				p.Parent = workspace
				table.insert(CageParts, { part = p, offset = off })
			end
		end
		for _, item in ipairs(CageParts) do
			item.part.CFrame = targetHrp.CFrame + item.offset
		end
	elseif #CageParts > 0 then
		ClearPartsTable(CageParts)
	end
end

-- // Chat //
local function SendChatMessage(msg)
	pcall(function()
		if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
			local channel = TextChatService.TextChannels:FindFirstChild("RBXGeneral")
			if channel then
				channel:SendAsync(msg)
				return
			end
		end
		local chatEvents = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
		if chatEvents and chatEvents:FindFirstChild("SayMessageRequest") then
			chatEvents.SayMessageRequest:FireServer(msg, "All")
		end
	end)
end

-- // Discord //
local function JoinDiscord()
	local invCode = "mbAxv2MeTX"
	local invLink = "https://discord.gg/" .. invCode
	Notify("RISE SHOP👑", "Link do Discord copiado!", 4)
	if setclipboard then
		pcall(function() setclipboard(invLink) end)
	end
end

-- // Super Pulo //
UserInputService.JumpRequest:Connect(function()
	if SuperJumpEnabled then
		local _, myHrp, myHum = GetLocalChar()
		if myHrp and myHum and myHum.FloorMaterial ~= Enum.Material.Air then
			myHrp.AssemblyLinearVelocity = Vector3.new(myHrp.AssemblyLinearVelocity.X, SuperJumpPower, myHrp.AssemblyLinearVelocity.Z)
			myHrp.Velocity = Vector3.new(myHrp.Velocity.X, SuperJumpPower, myHrp.Velocity.Z)
		end
	end
end)

-- // Anti-AFK //
LocalPlayer.Idled:Connect(function()
	if AntiAFKEnabled then
		pcall(function()
			VirtualUser:CaptureController()
			VirtualUser:ClickButton2(Vector2.new())
		end)
	end
end)

-- // Auto-Rejoin seguro //
pcall(function()
	local promptGui = game:GetService("CoreGui"):FindFirstChild("RobloxPromptGui")
	if promptGui and promptGui:FindFirstChild("promptOverlay") then
		promptGui.promptOverlay.ChildAdded:Connect(function(child)
			if AutoRejoinEnabled and child.Name == "ErrorPrompt" then
				TeleportService:Teleport(game.PlaceId, LocalPlayer)
			end
		end)
	end
end)

LocalPlayer.CharacterAdded:Connect(function(char)
	BindCharacterTouch(char)
end)

if LocalPlayer.Character then
	BindCharacterTouch(LocalPlayer.Character)
end

-- // Loop Principal //
RunService.Heartbeat:Connect(function(dt)
	if AutoCycleTargetEnabled and tick() - lastCycleTime >= 2.5 then
		lastCycleTime = tick()
		local candidates = {}
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
				table.insert(candidates, p)
			end
		end
		if #candidates > 0 then
			local picked = candidates[math.random(1, #candidates)]
			SelectedTargetName = picked.Name
			SelectedTarget = picked
		end
	end

	local myChar, myHrp, myHum = GetLocalChar()
	local targetPlr, targetChar, targetHrp, targetHum = GetValidTarget()

	UpdateTargetStructures(targetHrp)

	if (TouchFlingEnabled or TouchPushEnabled) and myHrp then
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= LocalPlayer and p.Character then
				local pHrp = p.Character:FindFirstChild("HumanoidRootPart")
				if pHrp and (pHrp.Position - myHrp.Position).Magnitude <= 5.5 then
					OnTouchPlayer(p)
				end
			end
		end
	end

	-- Voo (Fly)
	if FlyEnabled and myHrp and myHum then
		myHum.PlatformStand = true
		local moveVec = Vector3.zero
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveVec += Camera.CFrame.LookVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveVec -= Camera.CFrame.LookVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveVec -= Camera.CFrame.RightVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveVec += Camera.CFrame.RightVector end
		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then moveVec += Vector3.new(0, 1, 0) end
		if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then moveVec -= Vector3.new(0, 1, 0) end

		if moveVec.Magnitude == 0 and myHum.MoveDirection.Magnitude > 0 then
			moveVec = Vector3.new(myHum.MoveDirection.X, Camera.CFrame.LookVector.Y * 0.4, myHum.MoveDirection.Z)
		end

		if moveVec.Magnitude > 0 then
			local vel = moveVec.Unit * FlySpeed
			myHrp.AssemblyLinearVelocity = vel
			myHrp.Velocity = vel
		else
			myHrp.AssemblyLinearVelocity = Vector3.zero
			myHrp.Velocity = Vector3.zero
		end
	elseif not FlyEnabled and myHum and myHum.PlatformStand and not PlatformStandEnabled then
		myHum.PlatformStand = false
	end

	-- Inverter Controles
	if ReverseMoveEnabled and myHum and myHum.MoveDirection.Magnitude > 0 then
		myHum:Move(-myHum.MoveDirection, false)
	end

	-- Atravessar Paredes
	if NoClipEnabled and myChar then
		for _, part in ipairs(myChar:GetDescendants()) do
			if part:IsA("BasePart") then
				part.CanCollide = false
			end
		end
	end

	-- Postura Deitada
	if PlatformStandEnabled and myHum then
		myHum.PlatformStand = true
	end

	-- Órbita
	OrbitAngle += OrbitSpeed * dt
	if OrbitEnabled and myHrp and targetHrp then
		local offsetX = math.cos(OrbitAngle) * OrbitRadius
		local offsetZ = math.sin(OrbitAngle) * OrbitRadius
		local orbitPos = targetHrp.Position + Vector3.new(offsetX, 1.5, offsetZ)
		myHrp.CFrame = CFrame.new(orbitPos, targetHrp.Position)
	end

	-- Modos de Grudar
	if AttachMode ~= "Nenhum" and myHrp and targetHrp then
		if AttachMode == "Cabeca" then
			myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 3.2, 0.3)
		elseif AttachMode == "Costas" then
			myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, 1.6)
		elseif AttachMode == "Frente" then
			myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, -1.8)
		elseif AttachMode == "Embaixo" then
			myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, -3.2, 0)
		end
		myHrp.AssemblyLinearVelocity = Vector3.zero
		myHrp.Velocity = Vector3.zero
	end

	-- Seguir em Loop
	if FollowLoopEnabled and myHrp and targetHrp and AttachMode == "Nenhum" and not OrbitEnabled then
		myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, 2.5)
	end

	-- Cercar Lados
	if SandwichEnabled and myHrp and targetHrp then
		SandwichSide = -SandwichSide
		myHrp.CFrame = targetHrp.CFrame * CFrame.new(2.2 * SandwichSide, 0, 0)
	end

	-- Spam de Pulo
	if SpamJumpEnabled and myHum then
		myHum.Jump = true
	end

	-- Spam de Sentar
	if SpamSitEnabled and myHum then
		myHum.Sit = not myHum.Sit
	end

	-- Loop de Dança
	if SpamDanceLoopEnabled and myHum then
		myHum.Sit = (math.floor(tick() * 4) % 2 == 0)
		myHum.Jump = (math.floor(tick() * 6) % 3 == 0)
	end

	-- Sentar em Qualquer Lugar
	if SitEverywhereEnabled and myHum then
		myHum.Sit = true
	end

	-- Spam Equipar Ferramenta
	if SpamToolEquipEnabled and myChar and myHum then
		local backpack = LocalPlayer:FindFirstChild("Backpack")
		if backpack then
			local tools = {}
			for _, item in ipairs(backpack:GetChildren()) do
				if item:IsA("Tool") then table.insert(tools, item) end
			end
			for _, item in ipairs(myChar:GetChildren()) do
				if item:IsA("Tool") then table.insert(tools, item) end
			end
			if #tools > 0 then
				local chosen = tools[math.random(1, #tools)]
				if chosen.Parent == backpack then
					myHum:EquipTool(chosen)
				else
					myHum:UnequipTools()
				end
			end
		end
	end

	-- Spam Ativar Ferramenta
	if SpamToolActivateEnabled and myChar then
		for _, tool in ipairs(myChar:GetChildren()) do
			if tool:IsA("Tool") then
				tool:Activate()
			end
		end
	end

	-- Congelar Posição Local
	if FreezePosEnabled and myHrp then
		if not FrozenCFrame then
			FrozenCFrame = myHrp.CFrame
		end
		myHrp.CFrame = FrozenCFrame
		myHrp.AssemblyLinearVelocity = Vector3.zero
		myHrp.AssemblyAngularVelocity = Vector3.zero
		myHrp.Velocity = Vector3.zero
		myHrp.RotVelocity = Vector3.zero
	else
		FrozenCFrame = nil
	end

	-- Girar Sem Parar Local
	if SpinEnabled and myHrp then
		myHrp.CFrame = myHrp.CFrame * CFrame.Angles(0, math.rad(SpinSpeed), 0)
	end

	-- Chat Spam
	if ChatSpamEnabled and tick() - lastChatTime >= ChatSpamDelay then
		lastChatTime = tick()
		SendChatMessage(ChatSpamText)
	end
end)

-- // INTERFACE KAVO //

local TargetTab = Window:NewTab("Alvo & Troll")

local SelectSec = TargetTab:NewSection("1. Seleção & Controle")
local playerDropdown = SelectSec:NewDropdown("Jogador Alvo", "Selecione o jogador ou 'Nenhum'", GetPlayerList(), function(selected)
	SelectedTargetName = selected
	if selected == "Nenhum" then
		ResetTargetAndNormal()
	else
		RefreshTarget()
	end
end)

SelectSec:NewButton("Atualizar Lista", "Recarrega os jogadores do servidor", function()
	playerDropdown:Refresh(GetPlayerList())
end)

SelectSec:NewToggle("Alternar Alvo Automático", "Troca o alvo selecionado sozinho", function(v)
	AutoCycleTargetEnabled = v
end)

SelectSec:NewButton("Remover Alvo (Normal)", "Desseleciona e restaura tudo ao normal", function()
	ResetTargetAndNormal()
	playerDropdown:Refresh(GetPlayerList())
end)

SelectSec:NewButton("PARAR TUDO", "Desativa todas as funções ativas", function()
	StopAllTrolls()
	playerDropdown:Refresh(GetPlayerList())
end)

local FlingSec = TargetTab:NewSection("2. Arremesso & Interação")
FlingSec:NewSlider("Força do Arremesso", "Potência do arremesso", 80000, 5000, function(v) FlingPower = v end)
FlingSec:NewButton("Arremessar Alvo", "Arremessa o jogador alvo", function()
	local targetPlr = GetValidTarget()
	if targetPlr then
		task.spawn(function() FlingPlayer(targetPlr) end)
	end
end)
FlingSec:NewButton("Arremessar Todos", "Arremessa todos do servidor", function()
	task.spawn(function()
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= LocalPlayer then
				FlingPlayer(p)
				task.wait(0.15)
			end
		end
	end)
end)
FlingSec:NewButton("Empurrar Alvo", "Empurra o alvo com impulso", function()
	local targetPlr = GetValidTarget()
	if targetPlr then PushPlayer(targetPlr) end
end)
FlingSec:NewToggle("Seguir em Loop", "Gruda atrás do alvo", function(v) FollowLoopEnabled = v end)
FlingSec:NewToggle("Cercar Lados", "Alterna lados no alvo", function(v) SandwichEnabled = v end)
FlingSec:NewToggle("Jaula em Volta do Alvo", "Fecha o alvo com blocos", function(v)
	CageEnabled = v
	if not v then ClearPartsTable(CageParts) end
end)
FlingSec:NewToggle("Blocos em Volta do Alvo", "Gira blocos físicos ao redor", function(v)
	SurroundBlocksEnabled = v
	if not v then ClearPartsTable(SurroundParts) end
end)

local AttachSec = TargetTab:NewSection("3. Grudar & Órbita")
AttachSec:NewButton("Teleportar até o Alvo", "Vai até o jogador", function()
	local _, _, targetHrp = GetValidTarget()
	local _, myHrp = GetLocalChar()
	if targetHrp and myHrp then myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, 3) end
end)
AttachSec:NewButton("Trocar Posição", "Troca de lugar com o alvo", function()
	local _, _, targetHrp = GetValidTarget()
	local _, myHrp = GetLocalChar()
	if targetHrp and myHrp then
		local temp = myHrp.CFrame
		myHrp.CFrame = targetHrp.CFrame
		targetHrp.CFrame = temp
	end
end)
AttachSec:NewDropdown("Modo de Grudar", "Posição fixa no alvo", {"Nenhum", "Cabeca", "Costas", "Frente", "Embaixo"}, function(selected)
	AttachMode = selected
end)
AttachSec:NewToggle("Órbita", "Gira em volta do alvo", function(v) OrbitEnabled = v end)
AttachSec:NewSlider("Raio da Órbita", "Distância do alvo", 25, 2, function(v) OrbitRadius = v end)
AttachSec:NewSlider("Velocidade da Órbita", "Velocidade do giro", 40, 2, function(v) OrbitSpeed = v end)

local MoveTab = Window:NewTab("Movimento & Toque")

local FlySec = MoveTab:NewSection("1. Voo & Física")
FlySec:NewToggle("Ativar Voo", "Voa na direção da câmera", function(v) FlyEnabled = v end)
FlySec:NewSlider("Velocidade do Voo", "Velocidade de voo", 250, 10, function(v) FlySpeed = v end)
FlySec:NewToggle("Super Pulo", "Aumenta a altura do pulo", function(v) SuperJumpEnabled = v end)
FlySec:NewSlider("Força do Super Pulo", "Altura do impulso", 300, 60, function(v) SuperJumpPower = v end)
FlySec:NewToggle("Atravessar Paredes", "Desativa colisão das partes", function(v) NoClipEnabled = v end)
FlySec:NewToggle("Deitar Personagem", "Desativa postura normal", function(v) PlatformStandEnabled = v end)
FlySec:NewToggle("Inverter Controles", "Inverte direção do movimento", function(v) ReverseMoveEnabled = v end)
FlySec:NewToggle("Anti-AFK", "Evita kick por inatividade", function(v) AntiAFKEnabled = v end)

local TouchSec = MoveTab:NewSection("2. Ao Encostar")
TouchSec:NewSlider("Força do Empurrão", "Força do empurrão ao encostar", 500, 50, function(v) PushPower = v end)
TouchSec:NewToggle("Encostar e Arremessar", "Arremessa quem encostar em você", function(v) TouchFlingEnabled = v end)
TouchSec:NewToggle("Encostar e Empurrar", "Empurra quem encostar em você", function(v) TouchPushEnabled = v end)

local ChaosTab = Window:NewTab("Caos & Animações")

local PhysSec = ChaosTab:NewSection("1. Física & Postura")
PhysSec:NewToggle("Girar Sem Parar", "Gira o personagem continuamente", function(v) SpinEnabled = v end)
PhysSec:NewSlider("Velocidade do Giro", "Velocidade de rotação", 120, 10, function(v) SpinSpeed = v end)
PhysSec:NewToggle("Congelar Posição", "Trava a posição atual", function(v) FreezePosEnabled = v end)

local AnimSec = ChaosTab:NewSection("2. Animações & Ferramentas")
AnimSec:NewToggle("Spam de Pulo", "Pula sem parar", function(v) SpamJumpEnabled = v end)
AnimSec:NewToggle("Spam de Sentar", "Senta e levanta rápido", function(v) SpamSitEnabled = v end)
AnimSec:NewToggle("Loop de Dança", "Alterna sentar e pular", function(v) SpamDanceLoopEnabled = v end)
AnimSec:NewToggle("Sentar em Qualquer Lugar", "Força postura sentada", function(v) SitEverywhereEnabled = v end)
AnimSec:NewToggle("Spam Equipar Ferramenta", "Equipa e desequipa ferramentas", function(v) SpamToolEquipEnabled = v end)
AnimSec:NewToggle("Spam Ativar Ferramenta", "Ativa ferramenta equipada em loop", function(v) SpamToolActivateEnabled = v end)
AnimSec:NewButton("Forçar Levantar", "Sai do estado sentado", function()
	local _, _, myHum = GetLocalChar()
	if myHum then
		myHum.Sit = false
		myHum.Jump = true
	end
end)
AnimSec:NewButton("Resetar Personagem", "Reseta o personagem na hora", function()
	local _, _, myHum = GetLocalChar()
	if myHum then myHum.Health = 0 end
end)

local ChatTab = Window:NewTab("Chat")
local ChatSec = ChatTab:NewSection("Mensagens")
ChatSec:NewTextBox("Texto do Spam", "Digite a mensagem", function(txt) ChatSpamText = txt end)
ChatSec:NewSlider("Intervalo (Segundos)", "Tempo entre mensagens", 15, 1, function(v) ChatSpamDelay = v end)
ChatSec:NewToggle("Ativar Spam no Chat", "Envia em loop", function(v) ChatSpamEnabled = v end)
ChatSec:NewButton("Enviar Agora", "Envia uma mensagem", function() SendChatMessage(ChatSpamText) end)

local ServerTab = Window:NewTab("Servidor & Config")
local ServerSec = ServerTab:NewSection("1. Conexão")
ServerSec:NewButton("Reentrar no Servidor", "Reconecta ao servidor atual", function()
	TeleportService:Teleport(game.PlaceId, LocalPlayer)
end)
ServerSec:NewButton("Trocar de Servidor", "Entra em outro servidor", function()
	pcall(function()
		local servers = HttpService:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100"))
		for _, s in pairs(servers.data) do
			if s.playing < s.maxPlayers and s.id ~= game.JobId then
				TeleportService:TeleportToPlaceInstance(game.PlaceId, s.id, LocalPlayer)
				break
			end
		end
	end)
end)
ServerSec:NewToggle("Auto Reentrar ao Ser Kickado", "Reconecta automaticamente", function(v) AutoRejoinEnabled = v end)

local ConfigSec = ServerTab:NewSection("2. Configurações")
ConfigSec:NewButton("Salvar Configurações", "Salva em arquivo JSON", function()
	local configData = {
		SelectedTargetName = SelectedTargetName,
		FlingPower = FlingPower,
		PushPower = PushPower,
		OrbitRadius = OrbitRadius,
		OrbitSpeed = OrbitSpeed,
		FlySpeed = FlySpeed,
		SuperJumpPower = SuperJumpPower,
		TouchFlingEnabled = TouchFlingEnabled,
		TouchPushEnabled = TouchPushEnabled,
		SpinSpeed = SpinSpeed,
		ChatSpamText = ChatSpamText,
		ChatSpamDelay = ChatSpamDelay,
		AutoRejoinEnabled = AutoRejoinEnabled,
		AntiAFKEnabled = AntiAFKEnabled
	}
	if writefile then
		pcall(function()
			writefile(ConfigFileName, HttpService:JSONEncode(configData))
			Notify("RISE SHOP👑", "Configuração salva!", 3)
		end)
	end
end)

ConfigSec:NewButton("Carregar Configurações", "Carrega do arquivo JSON", function()
	if isfile and isfile(ConfigFileName) and readfile then
		pcall(function()
			local data = HttpService:JSONDecode(readfile(ConfigFileName))
			SelectedTargetName = data.SelectedTargetName or SelectedTargetName
			FlingPower = data.FlingPower or FlingPower
			PushPower = data.PushPower or PushPower
			OrbitRadius = data.OrbitRadius or OrbitRadius
			OrbitSpeed = data.OrbitSpeed or OrbitSpeed
			FlySpeed = data.FlySpeed or FlySpeed
			SuperJumpPower = data.SuperJumpPower or SuperJumpPower
			TouchFlingEnabled = data.TouchFlingEnabled or TouchFlingEnabled
			TouchPushEnabled = data.TouchPushEnabled or TouchPushEnabled
			SpinSpeed = data.SpinSpeed or SpinSpeed
			ChatSpamText = data.ChatSpamText or ChatSpamText
			ChatSpamDelay = data.ChatSpamDelay or ChatSpamDelay
			AutoRejoinEnabled = data.AutoRejoinEnabled or AutoRejoinEnabled
			AntiAFKEnabled = data.AntiAFKEnabled or AntiAFKEnabled
			RefreshTarget()
			Notify("RISE SHOP👑", "Configuração carregada!", 3)
		end)
	end
end)

local InfoTab = Window:NewTab("Info")
local InfoSec = InfoTab:NewSection("RISE SHOP👑")
InfoSec:NewButton("Discord", "Copia o link do Discord", function() JoinDiscord() end)
InfoSec:NewLabel("Marca: RISE SHOP👑")
InfoSec:NewLabel("Painel em Português")

-- // Botão Flutuante "👑" no PlayerGui (sem mexer em CoreGui) //
local FloatingGui = Instance.new("ScreenGui")
FloatingGui.Name = "RiseShop_Floating"
FloatingGui.ResetOnSpawn = false
FloatingGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
FloatingGui.Parent = PlayerGui

local MainButton = Instance.new("TextButton")
MainButton.Name = "MainButton"
MainButton.Parent = FloatingGui
MainButton.BackgroundColor3 = Color3.fromRGB(35, 39, 46)
MainButton.Position = UDim2.new(0.08, 0, 0.25, 0)
MainButton.Size = UDim2.new(0, 62, 0, 62)
MainButton.Font = Enum.Font.GothamBold
MainButton.Text = "👑"
MainButton.TextColor3 = Color3.fromRGB(255, 255, 255)
MainButton.TextSize = 32
MainButton.Active = true
MainButton.Draggable = true

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(1, 0)
Corner.Parent = MainButton

local Stroke = Instance.new("UIStroke")
Stroke.Color = Color3.fromRGB(110, 118, 130)
Stroke.Thickness = 2
Stroke.Parent = MainButton

MainButton.MouseButton1Click:Connect(function()
	pcall(function()
		Kavo:ToggleUI()
	end)
end)

JoinDiscord()
