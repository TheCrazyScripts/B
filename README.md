# --========================================================--
--                     CRAZY HUB                         --
--========================================================--

repeat task.wait() until game:IsLoaded()

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local VirtualUser = game:GetService("VirtualUser")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

--========================================================--
-- SISTEMA DE CONFIGURAÇÃO
--========================================================--

local ConfigFileName = "CrazyHubConfig.json"

local MEU_SCRIPT_LOADSTRING = [[
loadstring(game:HttpGet("https://pastebin.com/raw/jQtgaHgw"))()
]]

local ConfigLoaded = {}
local ConfigFileExists = false

local function LoadConfigFile()
	if not isfile or not readfile then
		return {}
	end

	local Success, Result = pcall(function()
		if not isfile(ConfigFileName) then
			return {}
		end

		ConfigFileExists = true

		local Content = readfile(ConfigFileName)

		if not Content or Content == "" then
			return {}
		end

		return HttpService:JSONDecode(Content)
	end)

	if Success and type(Result) == "table" then
		return Result
	end

	return {}
end

ConfigLoaded = LoadConfigFile()

--========================================================--
-- CONFIGURAÇÕES
--========================================================--

local M1Interval = tonumber(ConfigLoaded.M1Interval) or 0.35
local BloodDrinkInterval = tonumber(ConfigLoaded.BloodDrinkInterval) or 3
local IncendiaInterval = tonumber(ConfigLoaded.IncendiaInterval) or 3
local PainInterval = tonumber(ConfigLoaded.PainInterval) or 3
local QuestInterval = tonumber(ConfigLoaded.QuestInterval) or 2

local FarmDistance = tonumber(ConfigLoaded.FarmDistance) or 4

local MoneyCollectMode = ConfigLoaded.MoneyCollectMode or "Nearby"
local MoneyNearbyRadius = tonumber(ConfigLoaded.MoneyNearbyRadius) or 35
local MoneyMediumRadius = tonumber(ConfigLoaded.MoneyMediumRadius) or 100

--========================================================--
-- ESTADOS
--========================================================--

local AutoFarmEnabled = ConfigLoaded.AutoFarm == true
local AutoQuestEnabled = ConfigLoaded.AutoQuest ~= false
local AutoCollectMoneyEnabled = ConfigLoaded.AutoCollectMoney == true

local BloodDrinkEnabled = ConfigLoaded.BloodDrink == true
local IncendiaEnabled = ConfigLoaded.Incendia == true
local PainEnabled = ConfigLoaded.Pain == true

local AttackMode = ConfigLoaded.AttackMode or "M1"
local FarmPosition = ConfigLoaded.FarmPosition or "Front"

local SelectedNPC = ConfigLoaded.SelectedNPC

local AutoExecuteEnabled = ConfigLoaded.AutoExecute == true
local SaveEnabled = ConfigLoaded.SaveEnabled == true

-- NOVO
local AntiAfkEnabled = ConfigLoaded.AntiAfk == true

local CurrentNPC = nil

local LastM1 = 0
local LastBloodDrink = 0
local LastIncendia = 0
local LastPain = 0
local LastQuest = 0

--========================================================--
-- ANTI AFK
--========================================================--

local AntiAfkConnection = nil

local function StartAntiAfk()

	if AntiAfkConnection then
		return
	end

	AntiAfkConnection = Player.Idled:Connect(function()

		if not AntiAfkEnabled then
			return
		end

		pcall(function()

			VirtualUser:Button2Down(
				Vector2.new(0,0),
				workspace.CurrentCamera.CFrame
			)

			task.wait(1)

			VirtualUser:Button2Up(
				Vector2.new(0,0),
				workspace.CurrentCamera.CFrame
			)

		end)

	end)

	print("[Crazy Hub] Anti Afk ativado.")
end

local function StopAntiAfk()

	if AntiAfkConnection then
		AntiAfkConnection:Disconnect()
		AntiAfkConnection = nil
	end

	print("[Crazy Hub] Anti Afk desativado.")
end

if AntiAfkEnabled then
	StartAntiAfk()
end

--========================================================--
-- SAVE
--========================================================--

local function UpdateConfigTable()

	getgenv().Config = getgenv().Config or {}

	getgenv().Config.AutoFarm = AutoFarmEnabled
	getgenv().Config.AutoQuest = AutoQuestEnabled
	getgenv().Config.AutoCollectMoney = AutoCollectMoneyEnabled

	getgenv().Config.BloodDrink = BloodDrinkEnabled
	getgenv().Config.Incendia = IncendiaEnabled
	getgenv().Config.Pain = PainEnabled

	getgenv().Config.AttackMode = AttackMode
	getgenv().Config.FarmPosition = FarmPosition
	getgenv().Config.SelectedNPC = SelectedNPC

	getgenv().Config.M1Interval = M1Interval
	getgenv().Config.BloodDrinkInterval = BloodDrinkInterval
	getgenv().Config.IncendiaInterval = IncendiaInterval
	getgenv().Config.PainInterval = PainInterval
	getgenv().Config.QuestInterval = QuestInterval

	getgenv().Config.FarmDistance = FarmDistance

	getgenv().Config.MoneyCollectMode = MoneyCollectMode
	getgenv().Config.MoneyNearbyRadius = MoneyNearbyRadius
	getgenv().Config.MoneyMediumRadius = MoneyMediumRadius

	getgenv().Config.AutoExecute = AutoExecuteEnabled
	getgenv().Config.SaveEnabled = SaveEnabled

	-- NOVO
	getgenv().Config.AntiAfk = AntiAfkEnabled

	return getgenv().Config
end

local function SaveConfig()

	if not writefile then
		warn(
			"[Crazy Hub] writefile não está disponível neste ambiente."
		)
		return false
	end

	local Config = UpdateConfigTable()

	local Success, Error = pcall(function()

		local JSON =
			HttpService:JSONEncode(Config)

		writefile(
			ConfigFileName,
			JSON
		)

	end)

	if not Success then

		warn(
			"[Crazy Hub] Erro ao salvar: "
				.. tostring(Error)
		)

		return false
	end

	print(
		"[Crazy Hub] Configurações salvas."
	)

	return true
end

getgenv().Config =
	getgenv().Config or {}

UpdateConfigTable()

--========================================================--
-- AUTO EXECUTE
--========================================================--

local function GetQueueFunction()

	if queue_on_teleport then
		return queue_on_teleport
	end

	if syn
		and syn.queue_on_teleport then

		return syn.queue_on_teleport
	end

	return nil
end

local function SetupAutoExecute()

	if not AutoExecuteEnabled then
		return false
	end

	local QueueFunction =
		GetQueueFunction()

	if not QueueFunction then

		warn(
			"[Crazy Hub] Este ambiente não possui queue_on_teleport."
		)

		return false
	end

	if not MEU_SCRIPT_LOADSTRING
		or MEU_SCRIPT_LOADSTRING == "" then

		return false
	end

	local Success =
		pcall(function()

			QueueFunction(
				MEU_SCRIPT_LOADSTRING
			)

		end)

	return Success
end

local function StartAutoFarm()

	AutoFarmEnabled = true
	CurrentNPC = nil

end

--========================================================--
-- REMOTES
--========================================================--

local CombatRemote
local BloodDrinkRequest
local IncendiaRequest
local WitchLifeDrainRequest
local HunterQuestRemote

pcall(function()

	CombatRemote =
		ReplicatedStorage
		:WaitForChild("Funções")
		:WaitForChild("Game15Fists")
		:WaitForChild("CombatRemote")

end)

pcall(function()

	BloodDrinkRequest =
		ReplicatedStorage
		:WaitForChild("Network")
		:WaitForChild("Combat")
		:WaitForChild("BloodDrinkRequest")

end)

pcall(function()

	IncendiaRequest =
		ReplicatedStorage
		:WaitForChild("Network")
		:WaitForChild("Combat")
		:WaitForChild("IncendiaRequest")

end)

pcall(function()

	WitchLifeDrainRequest =
		ReplicatedStorage
		:WaitForChild("Network")
		:WaitForChild("Combat")
		:WaitForChild("WitchLifeDrainRequest")

end)

pcall(function()

	HunterQuestRemote =
		ReplicatedStorage
		:WaitForChild("HunterQuestRemotes")
		:WaitForChild("HunterQuestRemote")

end)

--========================================================--
-- CHARACTER
--========================================================--

local function GetCharacter()
	return Player.Character
end

local function GetHumanoid()

	local Character =
		GetCharacter()

	if not Character then
		return nil
	end

	return Character:FindFirstChildOfClass(
		"Humanoid"
	)

end

local function GetRoot()

	local Character =
		GetCharacter()

	if not Character then
		return nil
	end

	return Character:FindFirstChild(
		"HumanoidRootPart"
	)

end

--========================================================--
-- FISTS
--========================================================--

local function EquipFists()

	local Character =
		GetCharacter()

	local Humanoid =
		GetHumanoid()

	if not Character
		or not Humanoid then

		return
	end

	local Fists =
		Character:FindFirstChild("Fists")

	if Fists
		and Fists:IsA("Tool") then

		return Fists
	end

	local Backpack =
		Player:FindFirstChildOfClass(
			"Backpack"
		)

	if Backpack then

		Fists =
			Backpack:FindFirstChild("Fists")

		if Fists
			and Fists:IsA("Tool") then

			Humanoid:EquipTool(
				Fists
			)

			return Fists
		end
	end
end

--========================================================--
-- NORMALIZAÇÃO DOS NPCS
--========================================================--

local function NormalizeNPCName(Name)

	if not Name then
		return ""
	end

	local UpperName =
		string.upper(Name)

	if string.find(
		UpperName,
		"HUNTER",
		1,
		true
	) then

		return "Hunter"
	end

	if string.find(
		UpperName,
		"GUARD",
		1,
		true
	) then

		return "Guard"
	end

	local Base =
		Name:match(
			"^(.-)%d+$"
		)

	if Base
		and Base ~= "" then

		return Base
	end

	return Name
end

--========================================================--
-- HUNTER LEVEL
--========================================================--

local function GetHunterLevel(NPC)

	if not NPC then
		return nil
	end

	local Head =
		NPC:FindFirstChild(
			"Head",
			true
		)

	if not Head then
		return nil
	end

	local Billboard =
		Head:FindFirstChild(
			"HunterLevelBillboard"
		)

	if not Billboard then
		return nil
	end

	local Label =
		Billboard:FindFirstChild(
			"TextLabel"
		)

	if not Label then
		return nil
	end

	return tonumber(
		tostring(
			Label.Text
		):match("%d+")
	)
end

--========================================================--
-- HUNTER TIER
--========================================================--

local function GetHunterTier(Level)

	if not Level then
		return "Hunter"
	end

	if Level < 100 then
		return "Hunter < 100"
	elseif Level < 300 then
		return "Hunter 100–299"
	elseif Level < 500 then
		return "Hunter 300–499"
	elseif Level < 800 then
		return "Hunter 500–799"
	elseif Level < 1000 then
		return "Hunter 800–999"
	elseif Level < 3000 then
		return "Hunter 1000–2999"
	elseif Level < 4000 then
		return "Hunter 3000–3999"
	elseif Level < 5000 then
		return "Hunter 4000–4999"
	else

		local Start =
			math.floor(
				Level / 1000
			) * 1000

		return "Hunter "
			.. Start
			.. "–"
			.. Start + 999
	end
end

--========================================================--
-- CATEGORIA NPC
--========================================================--

local function GetNPCCategory(NPC)

	if not NPC then
		return nil
	end

	local HumansFolder =
		workspace:FindFirstChild("Humans")

	if HumansFolder
		and NPC:IsDescendantOf(
			HumansFolder
		) then

		return "Humans"
	end

	local UpperName =
		string.upper(NPC.Name)

	if string.find(
		UpperName,
		"HUNTER",
		1,
		true
	) then

		return GetHunterTier(
			GetHunterLevel(NPC)
		)
	end

	return NormalizeNPCName(
		NPC.Name
	)
end

--========================================================--
-- NPC HELPERS
--========================================================--

local function GetNPCRoot(NPC)

	if not NPC then
		return nil
	end

	return NPC:FindFirstChild(
		"HumanoidRootPart",
		true
	)
	or NPC.PrimaryPart
	or NPC:FindFirstChildWhichIsA(
		"BasePart",
		true
	)

end

local function GetNPCHumanoid(NPC)

	if not NPC then
		return nil
	end

	return NPC:FindFirstChildOfClass(
		"Humanoid"
	)

end

local function IsNPCAlive(NPC)

	local Humanoid =
		GetNPCHumanoid(NPC)

	return Humanoid
		and Humanoid.Health > 0
end

local function IsInsideHumansFolder(Object)

	local HumansFolder =
		workspace:FindFirstChild("Humans")

	if HumansFolder
		and Object:IsDescendantOf(
			HumansFolder
		) then

		return true
	end

	local Current = Object

	while Current
		and Current ~= workspace do

		if Current:IsA("Folder")
			and string.lower(
				Current.Name
			) == "humans" then

			return true
		end

		Current =
			Current.Parent
	end

	return false
end

local function GetAllNPCs()

	local Result = {}
	local Added = {}

	local NPCFolder =
		workspace:FindFirstChild("NPCS")

	local HumansFolder =
		workspace:FindFirstChild("Humans")

	local function AddNPC(Object)

		if not Object:IsA("Model") then
			return
		end

		if Added[Object] then
			return
		end

		if IsInsideHumansFolder(Object) then
			return
		end

		if not Object:FindFirstChildOfClass(
			"Humanoid"
		) then

			return
		end

		Added[Object] = true

		table.insert(
			Result,
			Object
		)
	end

	if NPCFolder then

		for _, Object in ipairs(
			NPCFolder:GetDescendants()
		) do

			AddNPC(Object)

		end
	end

	if HumansFolder then

		for _, Object in ipairs(
			HumansFolder:GetDescendants()
		) do

			if Object:IsA("Model")
				and Object:FindFirstChildOfClass(
					"Humanoid"
				)
				and not Added[Object] then

				Added[Object] = true

				table.insert(
					Result,
					Object
				)

			end
		end
	end

	return Result
end

local function FindNPC(Category)

	if not Category then
		return nil
	end

	for _, NPC in ipairs(
		GetAllNPCs()
	) do

		if GetNPCCategory(NPC)
			== Category then

			local Humanoid =
				GetNPCHumanoid(NPC)

			if Humanoid
				and Humanoid.Health > 0 then

				return NPC
			end
		end
	end

	return nil
end

--========================================================--
-- POSIÇÃO
--========================================================--

local function GetFarmCFrame(NPCRoot)

	if not NPCRoot then
		return nil
	end

	local Position =
		NPCRoot.Position

	if FarmPosition == "Front" then

		return CFrame.lookAt(
			Position
				+ NPCRoot.CFrame.LookVector
				* FarmDistance,
			Position
		)

	elseif FarmPosition == "Back" then

		return CFrame.lookAt(
			Position
				- NPCRoot.CFrame.LookVector
				* FarmDistance,
			Position
		)

	elseif FarmPosition == "Side" then

		return CFrame.lookAt(
			Position
				+ NPCRoot.CFrame.RightVector
				* FarmDistance,
			Position
		)

	elseif FarmPosition == "Above" then

		return CFrame.lookAt(
			Position
				+ Vector3.new(
					0,
					FarmDistance,
					0
				),
			Position
		)
	end
end

local function LockPlayerToNPC(NPC)

	local Root =
		GetRoot()

	local NPCRoot =
		GetNPCRoot(NPC)

	if not Root
		or not NPCRoot then

		return
	end

	local Position =
		GetFarmCFrame(NPCRoot)

	if Position then
		Root.CFrame = Position
	end
end

--========================================================--
-- M1
--========================================================--

local function DoM1()

	if not CombatRemote then
		return
	end

	local Now =
		os.clock()

	if Now - LastM1
		< M1Interval then

		return
	end

	LastM1 = Now

	EquipFists()

	pcall(function()

		CombatRemote:FireServer(
			"Attack"
		)

	end)
end

--========================================================--
-- BLOOD DRINK
--========================================================--

local function DoBloodDrink(NPC)

	if not BloodDrinkEnabled
		or not BloodDrinkRequest
		or not NPC then

		return
	end

	local Now =
		os.clock()

	if Now - LastBloodDrink
		< BloodDrinkInterval then

		return
	end

	LastBloodDrink = Now

	pcall(function()

		BloodDrinkRequest:FireServer(
			"Begin",
			"UI"
		)

		BloodDrinkRequest:FireServer(
			"Bite",
			NPC,
			"UI"
		)

	end)
end

--========================================================--
-- INCENDIA
--========================================================--

local function DoIncendia(NPC)

	if not IncendiaEnabled
		or not IncendiaRequest then

		return
	end

	local NPCRoot =
		GetNPCRoot(NPC)

	if not NPCRoot then
		return
	end

	local Now =
		os.clock()

	if Now - LastIncendia
		< IncendiaInterval then

		return
	end

	LastIncendia = Now

	pcall(function()

		IncendiaRequest:FireServer(
			"Cast",
			NPCRoot.Position,
			NPC
		)

	end)
end

--========================================================--
-- PAIN
--========================================================--

local function DoPain()

	if not PainEnabled
		or not WitchLifeDrainRequest then

		return
	end

	local Now =
		os.clock()

	if Now - LastPain
		< PainInterval then

		return
	end

	LastPain = Now

	pcall(function()

		WitchLifeDrainRequest:FireServer()

	end)
end

--========================================================--
-- QUEST
--========================================================--

local function DoQuest()

	if not AutoQuestEnabled
		or not HunterQuestRemote then

		return
	end

	local Now =
		os.clock()

	if Now - LastQuest
		< QuestInterval then

		return
	end

	LastQuest = Now

	pcall(function()

		HunterQuestRemote:FireServer(
			"SetAutoRepeat",
			true
		)

	end)
end

--========================================================--
-- MONEY
--========================================================--

local function GetMoneyFolder()

	return workspace:FindFirstChild(
		"DroppedMoney"
	)

end

local function GetMoneyBagRoot(Bag)

	if not Bag then
		return nil
	end

	if Bag:IsA("BasePart") then
		return Bag
	end

	if Bag:IsA("Model") then

		return Bag.PrimaryPart
			or Bag:FindFirstChildWhichIsA(
				"BasePart",
				true
			)

	end

	return Bag:FindFirstChildWhichIsA(
		"BasePart",
		true
	)
end

local function GetMoneyBags()

	local Folder =
		GetMoneyFolder()

	if not Folder then
		return {}
	end

	local Bags = {}

	for _, Object in ipairs(
		Folder:GetDescendants()
	) do

		if string.lower(
			Object.Name
		) == "killmoneybag" then

			if GetMoneyBagRoot(Object) then

				table.insert(
					Bags,
					Object
				)

			end
		end
	end

	return Bags
end

local function GetMoneyRadius()

	if MoneyCollectMode == "Nearby" then
		return MoneyNearbyRadius
	end

	if MoneyCollectMode == "Medium" then
		return MoneyMediumRadius
	end

	return math.huge
end

local function FindNearestMoneyBag()

	local PlayerRoot =
		GetRoot()

	if not PlayerRoot then
		return nil
	end

	local Nearest
	local NearestDistance =
		math.huge

	for _, Bag in ipairs(
		GetMoneyBags()
	) do

		if Bag.Parent then

			local BagRoot =
				GetMoneyBagRoot(Bag)

			if BagRoot then

				local Distance =
					(
						BagRoot.Position
						- PlayerRoot.Position
					).Magnitude

				if Distance <= GetMoneyRadius()
					and Distance < NearestDistance then

					Nearest = Bag
					NearestDistance = Distance

				end
			end
		end
	end

	return Nearest
end

local function CollectMoneyBag(Bag)

	if not Bag
		or not Bag.Parent then

		return false
	end

	local Root =
		GetRoot()

	local BagRoot =
		GetMoneyBagRoot(Bag)

	if not Root
		or not BagRoot then

		return false
	end

	Root.CFrame =
		BagRoot.CFrame
		+ Vector3.new(
			0,
			1.5,
			0
		)

	task.wait(0.12)

	return true
end

local function AutoCollectMoney()

	if not AutoCollectMoneyEnabled then
		return
	end

	local Bag =
		FindNearestMoneyBag()

	if Bag then
		CollectMoneyBag(Bag)
	end
end

--========================================================--
-- ATAQUES
--========================================================--

local function DoAttacks(NPC)

	if AttackMode == "M1" then

		DoM1()

	elseif AttackMode == "Powers" then

		DoBloodDrink(NPC)
		DoIncendia(NPC)
		DoPain()

	elseif AttackMode == "Both" then

		DoM1()
		DoBloodDrink(NPC)
		DoIncendia(NPC)
		DoPain()

	end
end

--========================================================--
-- GUI
--========================================================--

local OldGui =
	PlayerGui:FindFirstChild(
		"CrazyHub"
	)

if OldGui then
	OldGui:Destroy()
end

local Gui =
	Instance.new("ScreenGui")

Gui.Name = "CrazyHub"
Gui.ResetOnSpawn = false
Gui.ZIndexBehavior =
	Enum.ZIndexBehavior.Sibling

Gui.Parent = PlayerGui

local Main =
	Instance.new("Frame")

Main.Size =
	UDim2.new(0,470,0,330)

Main.Position =
	UDim2.new(0.5,-235,0.5,-165)

Main.BackgroundColor3 =
	Color3.fromRGB(14,14,17)

Main.BorderSizePixel = 0
Main.ClipsDescendants = false
Main.Parent = Gui

local MainCorner =
	Instance.new("UICorner")

MainCorner.CornerRadius =
	UDim.new(0,8)

MainCorner.Parent = Main

--========================================================--
-- TOP
--========================================================--

local Top =
	Instance.new("Frame")

Top.Size =
	UDim2.new(1,0,0,40)

Top.BackgroundColor3 =
	Color3.fromRGB(17,17,20)

Top.BorderSizePixel = 0
Top.Parent = Main

local TopCorner =
	Instance.new("UICorner")

TopCorner.CornerRadius =
	UDim.new(0,8)

TopCorner.Parent = Top

local Title =
	Instance.new("TextLabel")

Title.Size =
	UDim2.new(1,-80,1,0)

Title.Position =
	UDim2.new(0,13,0,0)

Title.BackgroundTransparency = 1
Title.Text = "[ CRAZY HUB ]"

Title.TextColor3 =
	Color3.fromRGB(45,255,120)

Title.TextSize = 15
Title.Font =
	Enum.Font.GothamBold

Title.TextXAlignment =
	Enum.TextXAlignment.Left

Title.Parent = Top

local Minimize =
	Instance.new("TextButton")

Minimize.Size =
	UDim2.new(0,30,0,34)

Minimize.Position =
	UDim2.new(1,-65,0,2)

Minimize.BackgroundTransparency = 1
Minimize.Text = "−"

Minimize.TextColor3 =
	Color3.fromRGB(200,200,200)

Minimize.TextSize = 20
Minimize.Font =
	Enum.Font.GothamBold

Minimize.Parent = Top

local Close =
	Instance.new("TextButton")

Close.Size =
	UDim2.new(0,30,0,34)

Close.Position =
	UDim2.new(1,-33,0,2)

Close.BackgroundTransparency = 1
Close.Text = "×"

Close.TextColor3 =
	Color3.fromRGB(240,240,240)

Close.TextSize = 20
Close.Font =
	Enum.Font.GothamBold

Close.Parent = Top

--========================================================--
-- SEARCH
--========================================================--

local Search =
	Instance.new("TextBox")

Search.Size =
	UDim2.new(1,-24,0,32)

Search.Position =
	UDim2.new(0,12,0,48)

Search.BackgroundColor3 =
	Color3.fromRGB(23,23,27)

Search.BorderSizePixel = 0
Search.PlaceholderText = "Pesquisar..."

Search.PlaceholderColor3 =
	Color3.fromRGB(100,100,105)

Search.Text = ""

Search.TextColor3 =
	Color3.fromRGB(220,220,225)

Search.TextSize = 12
Search.Font =
	Enum.Font.Gotham

Search.ClearTextOnFocus = false
Search.Parent = Main

local SearchCorner =
	Instance.new("UICorner")

SearchCorner.CornerRadius =
	UDim.new(0,5)

SearchCorner.Parent = Search

local SearchPadding =
	Instance.new("UIPadding")

SearchPadding.PaddingLeft =
	UDim.new(0,35)

SearchPadding.Parent = Search

local SearchIcon =
	Instance.new("TextLabel")

SearchIcon.Size =
	UDim2.new(0,25,0,32)

SearchIcon.Position =
	UDim2.new(0,15,0,48)

SearchIcon.BackgroundTransparency = 1
SearchIcon.Text = "⌕"

SearchIcon.TextColor3 =
	Color3.fromRGB(130,130,135)

SearchIcon.TextSize = 20
SearchIcon.Font =
	Enum.Font.Gotham

SearchIcon.ZIndex = 3
SearchIcon.Parent = Main

--========================================================--
-- SIDEBAR
--========================================================--

local Sidebar =
	Instance.new("Frame")

Sidebar.Size =
	UDim2.new(0,145,1,-94)

Sidebar.Position =
	UDim2.new(0,0,0,94)

Sidebar.BackgroundColor3 =
	Color3.fromRGB(15,15,18)

Sidebar.BorderSizePixel = 0
Sidebar.Parent = Main

local SidebarLayout =
	Instance.new("UIListLayout")

SidebarLayout.Padding =
	UDim.new(0,3)

SidebarLayout.Parent = Sidebar

local SidebarPadding =
	Instance.new("UIPadding")

SidebarPadding.PaddingTop =
	UDim.new(0,7)

SidebarPadding.PaddingLeft =
	UDim.new(0,6)

SidebarPadding.PaddingRight =
	UDim.new(0,6)

SidebarPadding.Parent = Sidebar

--========================================================--
-- CONTENT
--========================================================--

local Content =
	Instance.new("Frame")

Content.Size =
	UDim2.new(1,-145,1,-94)

Content.Position =
	UDim2.new(0,145,0,94)

Content.BackgroundColor3 =
	Color3.fromRGB(17,17,20)

Content.BorderSizePixel = 0
Content.ClipsDescendants = true
Content.Parent = Main

--========================================================--
-- DRAG
--========================================================--

local Dragging = false
local DragStart
local StartPosition

Top.InputBegan:Connect(function(Input)

	if Input.UserInputType ==
		Enum.UserInputType.MouseButton1
		or Input.UserInputType ==
		Enum.UserInputType.Touch then

		Dragging = true
		DragStart = Input.Position
		StartPosition = Main.Position

	end
end)

UserInputService.InputChanged:Connect(function(Input)

	if not Dragging then
		return
	end

	if Input.UserInputType ~=
		Enum.UserInputType.MouseMovement
		and Input.UserInputType ~=
		Enum.UserInputType.Touch then

		return
	end

	local Delta =
		Input.Position - DragStart

	Main.Position =
		UDim2.new(
			StartPosition.X.Scale,
			StartPosition.X.Offset + Delta.X,
			StartPosition.Y.Scale,
			StartPosition.Y.Offset + Delta.Y
		)
end)

UserInputService.InputEnded:Connect(function(Input)

	if Input.UserInputType ==
		Enum.UserInputType.MouseButton1
		or Input.UserInputType ==
		Enum.UserInputType.Touch then

		Dragging = false

	end
end)

--========================================================--
-- MINIMIZAR
--========================================================--

local Minimized = false

Minimize.MouseButton1Click:Connect(function()

	Minimized = not Minimized

	Search.Visible = not Minimized
	SearchIcon.Visible = not Minimized
	Sidebar.Visible = not Minimized
	Content.Visible = not Minimized

	if Minimized then

		Main.Size =
			UDim2.new(0,470,0,40)

	else

		Main.Size =
			UDim2.new(0,470,0,330)

	end
end)

Close.MouseButton1Click:Connect(function()
	Gui:Destroy()
end)

--========================================================--
-- PÁGINAS
--========================================================--

local Pages = {}
local NavigationButtons = {}

local function CreatePage(Name)

	local Page =
		Instance.new("ScrollingFrame")

	Page.Name = Name

	Page.Size =
		UDim2.new(1,-18,1,-14)

	Page.Position =
		UDim2.new(0,9,0,7)

	Page.BackgroundTransparency = 1
	Page.BorderSizePixel = 0
	Page.ScrollBarThickness = 4

	Page.ScrollBarImageColor3 =
		Color3.fromRGB(70,70,75)

	Page.ScrollingDirection =
		Enum.ScrollingDirection.Y

	Page.AutomaticCanvasSize =
		Enum.AutomaticSize.Y

	Page.CanvasSize =
		UDim2.new(0,0,0,0)

	Page.Visible = false
	Page.Parent = Content

	local Layout =
		Instance.new("UIListLayout")

	Layout.Padding =
		UDim.new(0,7)

	Layout.Parent = Page

	local Padding =
		Instance.new("UIPadding")

	Padding.PaddingBottom =
		UDim.new(0,12)

	Padding.Parent = Page

	Pages[Name] = Page

	return Page
end

local HomePage = CreatePage("Home")
local NPCPage = CreatePage("NPC Farm")
local TeleportPage = CreatePage("Teleports")
local PowersPage = CreatePage("Powers")
local SettingsPage = CreatePage("Settings")

--========================================================--
-- NAVEGAÇÃO
--========================================================--

local function CreateNav(Name,Text,Order)

	local Button =
		Instance.new("TextButton")

	Button.Name = Name

	Button.Size =
		UDim2.new(1,0,0,34)

	Button.BackgroundColor3 =
		Color3.fromRGB(15,15,18)

	Button.BorderSizePixel = 0
	Button.Text = Text

	Button.TextColor3 =
		Color3.fromRGB(150,150,155)

	Button.TextSize = 12
	Button.Font =
		Enum.Font.GothamMedium

	Button.TextXAlignment =
		Enum.TextXAlignment.Left

	Button.LayoutOrder = Order
	Button.Parent = Sidebar

	local Padding =
		Instance.new("UIPadding")

	Padding.PaddingLeft =
		UDim.new(0,10)

	Padding.Parent = Button

	local Corner =
		Instance.new("UICorner")

	Corner.CornerRadius =
		UDim.new(0,5)

	Corner.Parent = Button

	NavigationButtons[Name] =
		Button

	return Button
end

local HomeNav = CreateNav("Home","◉  Home",1)
local NPCNav = CreateNav("NPC Farm","◉  NPC Farm",2)
local TeleportNav = CreateNav("Teleports","✚  Teleports",3)
local PowersNav = CreateNav("Powers","⚡  Powers",4)
local SettingsNav = CreateNav("Settings","⚙  Settings",5)

local function ShowPage(Name)

	for PageName, Page in pairs(Pages) do
		Page.Visible = PageName == Name
	end

	for ButtonName, Button in pairs(NavigationButtons) do

		if ButtonName == Name then

			Button.BackgroundColor3 =
				Color3.fromRGB(25,80,48)

			Button.TextColor3 =
				Color3.fromRGB(45,255,120)

		else

			Button.BackgroundColor3 =
				Color3.fromRGB(15,15,18)

			Button.TextColor3 =
				Color3.fromRGB(150,150,155)

		end
	end
end

HomeNav.MouseButton1Click:Connect(function()
	ShowPage("Home")
end)

NPCNav.MouseButton1Click:Connect(function()
	ShowPage("NPC Farm")
end)

TeleportNav.MouseButton1Click:Connect(function()
	ShowPage("Teleports")
end)

PowersNav.MouseButton1Click:Connect(function()
	ShowPage("Powers")
end)

SettingsNav.MouseButton1Click:Connect(function()
	ShowPage("Settings")
end)

--========================================================--
-- UI HELPERS
--========================================================--

local function CreateSection(Page,Text)

	local Label =
		Instance.new("TextLabel")

	Label.Size =
		UDim2.new(1,0,0,25)

	Label.BackgroundTransparency = 1
	Label.Text = Text

	Label.TextColor3 =
		Color3.fromRGB(220,220,225)

	Label.TextSize = 14
	Label.Font =
		Enum.Font.GothamBold

	Label.TextXAlignment =
		Enum.TextXAlignment.Left

	Label.Parent = Page

	return Label
end

local function CreateBox(Page,Text)

	local Button =
		Instance.new("TextButton")

	Button.Size =
		UDim2.new(1,-4,0,37)

	Button.BackgroundColor3 =
		Color3.fromRGB(24,24,28)

	Button.BorderSizePixel = 0
	Button.Text = Text

	Button.TextColor3 =
		Color3.fromRGB(190,190,195)

	Button.TextSize = 12
	Button.Font =
		Enum.Font.GothamMedium

	Button.TextXAlignment =
		Enum.TextXAlignment.Left

	Button.Parent = Page

	local Padding =
		Instance.new("UIPadding")

	Padding.PaddingLeft =
		UDim.new(0,11)

	Padding.Parent = Button

	local Corner =
		Instance.new("UICorner")

	Corner.CornerRadius =
		UDim.new(0,5)

	Corner.Parent = Button

	return Button
end

--========================================================--
-- TOGGLE
--========================================================--

local function CreateToggle(Page,Text,Default,Callback)

	local Button =
		CreateBox(
			Page,
			Text .. "                         [OFF]"
		)

	local State = Default

	local function Update()

		if State then

			Button.Text =
				Text .. "                         [ON]"

			Button.TextColor3 =
				Color3.fromRGB(
					45,255,120
				)

		else

			Button.Text =
				Text .. "                         [OFF]"

			Button.TextColor3 =
				Color3.fromRGB(
					190,190,195
				)

		end
	end

	Button.MouseButton1Click:Connect(function()

		State = not State

		Update()

		Callback(State)

	end)

	Update()

	return Button
end

--========================================================--
-- DROPDOWN
--========================================================--

local OpenDropdown = nil

local function CreateDropdown(
	Page,
	TitleText,
	Options,
	Default,
	Callback
)

	local Holder =
		Instance.new("Frame")

	Holder.Size =
		UDim2.new(1,-4,0,37)

	Holder.BackgroundTransparency = 1
	Holder.BorderSizePixel = 0
	Holder.ClipsDescendants = false
	Holder.Parent = Page

	local Box =
		Instance.new("Frame")

	Box.Size =
		UDim2.new(1,0,0,37)

	Box.BackgroundColor3 =
		Color3.fromRGB(24,24,28)

	Box.BorderSizePixel = 0
	Box.ZIndex = 10
	Box.Parent = Holder

	local Corner =
		Instance.new("UICorner")

	Corner.CornerRadius =
		UDim.new(0,5)

	Corner.Parent = Box

	local Button =
		Instance.new("TextButton")

	Button.Size =
		UDim2.new(1,0,1,0)

	Button.BackgroundTransparency = 1

	Button.Text =
		TitleText
		.. ": "
		.. tostring(Default)

	Button.TextColor3 =
		Color3.fromRGB(195,195,200)

	Button.TextSize = 12
	Button.Font =
		Enum.Font.GothamMedium

	Button.TextXAlignment =
		Enum.TextXAlignment.Left

	Button.ZIndex = 11
	Button.Parent = Box

	local Padding =
		Instance.new("UIPadding")

	Padding.PaddingLeft =
		UDim.new(0,11)

	Padding.Parent = Button

	local Arrow =
		Instance.new("TextLabel")

	Arrow.Size =
		UDim2.new(0,25,1,0)

	Arrow.Position =
		UDim2.new(1,-30,0,0)

	Arrow.BackgroundTransparency = 1
	Arrow.Text = "▼"

	Arrow.TextColor3 =
		Color3.fromRGB(130,130,135)

	Arrow.TextSize = 10
	Arrow.ZIndex = 12
	Arrow.Parent = Box

	local List =
		Instance.new("ScrollingFrame")

	List.Name = "DropdownScrolling"

	List.Size =
		UDim2.new(1,0,0,120)

	List.Position =
		UDim2.new(0,0,0,40)

	List.BackgroundColor3 =
		Color3.fromRGB(21,21,25)

	List.BorderSizePixel = 0
	List.ScrollBarThickness = 4

	List.ScrollBarImageColor3 =
		Color3.fromRGB(70,70,75)

	List.AutomaticCanvasSize =
		Enum.AutomaticSize.Y

	List.CanvasSize =
		UDim2.new(0,0,0,0)

	List.Visible = false
	List.ZIndex = 100
	List.Parent = Holder

	local ListCorner =
		Instance.new("UICorner")

	ListCorner.CornerRadius =
		UDim.new(0,5)

	ListCorner.Parent = List

	local ListPadding =
		Instance.new("UIPadding")

	ListPadding.PaddingTop =
		UDim.new(0,4)

	ListPadding.PaddingBottom =
		UDim.new(0,4)

	ListPadding.Parent = List

	local Layout =
		Instance.new("UIListLayout")

	Layout.Padding =
		UDim.new(0,2)

	Layout.Parent = List

	for _, Option in ipairs(Options) do

		local OptionButton =
			Instance.new("TextButton")

		OptionButton.Size =
			UDim2.new(1,-7,0,30)

		OptionButton.BackgroundColor3 =
			Color3.fromRGB(27,27,32)

		OptionButton.BorderSizePixel = 0
		OptionButton.Text = tostring(Option)

		OptionButton.TextColor3 =
			Color3.fromRGB(185,185,190)

		OptionButton.TextSize = 11
		OptionButton.Font =
			Enum.Font.Gotham

		OptionButton.TextXAlignment =
			Enum.TextXAlignment.Left

		OptionButton.ZIndex = 101
		OptionButton.Parent = List

		local OptionPadding =
			Instance.new("UIPadding")

		OptionPadding.PaddingLeft =
			UDim.new(0,10)

		OptionPadding.Parent =
			OptionButton

		local OptionCorner =
			Instance.new("UICorner")

		OptionCorner.CornerRadius =
			UDim.new(0,4)

		OptionCorner.Parent =
			OptionButton

		OptionButton.MouseButton1Click:Connect(function()

			Button.Text =
				TitleText
				.. ": "
				.. tostring(Option)

			List.Visible = false

			Holder.Size =
				UDim2.new(1,-4,0,37)

			OpenDropdown = nil

			Callback(Option)

		end)
	end

	Button.MouseButton1Click:Connect(function()

		if List.Visible then

			List.Visible = false

			Holder.Size =
				UDim2.new(1,-4,0,37)

			OpenDropdown = nil

		else

			if OpenDropdown
				and OpenDropdown ~= Holder then

				local OldList =
					OpenDropdown:FindFirstChild(
						"DropdownScrolling"
					)

				if OldList then
					OldList.Visible = false
				end

				OpenDropdown.Size =
					UDim2.new(1,-4,0,37)

			end

			OpenDropdown = Holder
			List.Visible = true

			Holder.Size =
				UDim2.new(1,-4,0,165)

		end
	end)

	return Holder
end

--========================================================--
-- HOME
--========================================================--

CreateSection(HomePage,"Home")

local HomeText =
	Instance.new("TextLabel")

HomeText.Size =
	UDim2.new(1,-4,0,85)

HomeText.BackgroundTransparency = 1

HomeText.Text =
	"Welcome to Crazy Hub\n\n"
	.. "NPC Farm • Powers • Quests • Teleports\n\n"
	.. "Configure everything through the sidebar."

HomeText.TextColor3 =
	Color3.fromRGB(160,160,165)

HomeText.TextSize = 12
HomeText.Font =
	Enum.Font.Gotham

HomeText.TextWrapped = true
HomeText.TextXAlignment =
	Enum.TextXAlignment.Left

HomeText.TextYAlignment =
	Enum.TextYAlignment.Top

HomeText.Parent = HomePage

--========================================================--
-- NPC FARM
--========================================================--

CreateSection(
	NPCPage,
	"NPC Farm"
)

local NPCButton =
	CreateBox(
		NPCPage,
		"NPC Target: "
			.. tostring(
				SelectedNPC
				or "None"
			)
	)

local NPCList =
	Instance.new("ScrollingFrame")

NPCList.Name = "NPCScrolling"

NPCList.Size =
	UDim2.new(1,-4,0,130)

NPCList.BackgroundColor3 =
	Color3.fromRGB(21,21,25)

NPCList.BorderSizePixel = 0
NPCList.ScrollBarThickness = 4

NPCList.ScrollBarImageColor3 =
	Color3.fromRGB(70,70,75)

NPCList.AutomaticCanvasSize =
	Enum.AutomaticSize.Y

NPCList.CanvasSize =
	UDim2.new(0,0,0,0)

NPCList.Visible = false
NPCList.Parent = NPCPage

local NPCCorner =
	Instance.new("UICorner")

NPCCorner.CornerRadius =
	UDim.new(0,5)

NPCCorner.Parent = NPCList

local NPCLayout =
	Instance.new("UIListLayout")

NPCLayout.Padding =
	UDim.new(0,3)

NPCLayout.Parent = NPCList

local NPCPadding =
	Instance.new("UIPadding")

NPCPadding.PaddingTop =
	UDim.new(0,4)

NPCPadding.PaddingBottom =
	UDim.new(0,4)

NPCPadding.Parent = NPCList

local function RefreshNPCList()

	for _, Child in ipairs(
		NPCList:GetChildren()
	) do

		if Child:IsA("TextButton") then
			Child:Destroy()
		end
	end

	local Categories = {}

	for _, NPC in ipairs(
		GetAllNPCs()
	) do

		local Category =
			GetNPCCategory(NPC)

		if Category then
			Categories[Category] = true
		end
	end

	local Names = {}

	for Name in pairs(Categories) do
		table.insert(Names,Name)
	end

	table.sort(
		Names,
		function(A,B)

			if A == "Humans" then
				return false
			end

			if B == "Humans" then
				return true
			end

			return A < B
		end
	)

	for _, Category in ipairs(Names) do

		local Button =
			Instance.new("TextButton")

		Button.Size =
			UDim2.new(1,-7,0,30)

		Button.BackgroundColor3 =
			Color3.fromRGB(29,29,34)

		Button.BorderSizePixel = 0
		Button.Text = Category

		Button.TextColor3 =
			Color3.fromRGB(190,190,195)

		Button.TextSize = 11
		Button.Font =
			Enum.Font.GothamMedium

		Button.TextXAlignment =
			Enum.TextXAlignment.Left

		Button.Parent = NPCList

		local Padding =
			Instance.new("UIPadding")

		Padding.PaddingLeft =
			UDim.new(0,10)

		Padding.Parent = Button

		local Corner =
			Instance.new("UICorner")

		Corner.CornerRadius =
			UDim.new(0,4)

		Corner.Parent = Button

		Button.MouseButton1Click:Connect(function()

			SelectedNPC = Category
			CurrentNPC = nil

			NPCButton.Text =
				"NPC Target: "
				.. Category

			NPCList.Visible = false

		end)
	end
end

NPCButton.MouseButton1Click:Connect(function()

	NPCList.Visible =
		not NPCList.Visible

	if NPCList.Visible then
		RefreshNPCList()
	end

end)

RefreshNPCList()

--========================================================--
-- AUTO FARM
--========================================================--

CreateToggle(
	NPCPage,
	"Auto Farm",
	AutoFarmEnabled,
	function(State)

		AutoFarmEnabled = State
		CurrentNPC = nil

		getgenv().Config.AutoFarm =
			State

		if State then
			StartAutoFarm()
		end

		SaveConfig()

	end
)

CreateToggle(
	NPCPage,
	"Auto Quest",
	AutoQuestEnabled,
	function(State)

		AutoQuestEnabled = State

		getgenv().Config.AutoQuest =
			State

	end
)

CreateToggle(
	NPCPage,
	"Auto Collect Money",
	AutoCollectMoneyEnabled,
	function(State)

		AutoCollectMoneyEnabled =
			State

		getgenv().Config.AutoCollectMoney =
			State

	end
)

CreateDropdown(
	NPCPage,
	"Money Collect Range",
	{
		"Nearby",
		"Medium",
		"Map"
	},
	MoneyCollectMode,
	function(Value)
		MoneyCollectMode = Value
	end
)

CreateDropdown(
	NPCPage,
	"Attack Mode",
	{
		"M1",
		"Powers",
		"Both"
	},
	AttackMode,
	function(Value)
		AttackMode = Value
	end
)

CreateDropdown(
	NPCPage,
	"Farm Position",
	{
		"Front",
		"Back",
		"Side",
		"Above"
	},
	FarmPosition,
	function(Value)
		FarmPosition = Value
	end
)

--========================================================--
-- POWERS
--========================================================--

CreateSection(
	PowersPage,
	"Powers"
)

CreateToggle(
	PowersPage,
	"Blood Drink",
	BloodDrinkEnabled,
	function(State)
		BloodDrinkEnabled = State
	end
)

CreateToggle(
	PowersPage,
	"Incendia",
	IncendiaEnabled,
	function(State)
		IncendiaEnabled = State
	end
)

CreateToggle(
	PowersPage,
	"Pain",
	PainEnabled,
	function(State)
		PainEnabled = State
	end
)

--========================================================--
-- TELEPORTS
--========================================================--

CreateSection(
	TeleportPage,
	"Map & Dimensions"
)

local ItemNames = {}
local ItemFolder =
	workspace:FindFirstChild("ITENS")

if ItemFolder then

	for _, Item in ipairs(
		ItemFolder:GetChildren()
	) do

		table.insert(
			ItemNames,
			Item.Name
		)

	end

	table.sort(ItemNames)

end

if #ItemNames == 0 then
	ItemNames = {"None"}
end

CreateDropdown(
	TeleportPage,
	"Map Teleports",
	ItemNames,
	ItemNames[1],
	function(Value)

		if not ItemFolder then
			return
		end

		local Object =
			ItemFolder:FindFirstChild(
				Value
			)

		if not Object then
			return
		end

		local Root =
			GetRoot()

		if not Root then
			return
		end

		local Target

		if Object:IsA("BasePart") then

			Target = Object

		elseif Object:IsA("Model") then

			Target =
				Object.PrimaryPart
				or Object:FindFirstChildWhichIsA(
					"BasePart",
					true
				)

		end

		if Target then

			Root.CFrame =
				Target.CFrame
				+ Vector3.new(0,3,0)

		end
	end
)

--========================================================--
-- SPAWNS
--========================================================--

local SpawnNames = {}
local SpawnObjects = {}
local UsedPositions = {}

for _, Object in ipairs(
	workspace:GetDescendants()
) do

	local IsSpawn = false

	if Object:IsA("SpawnLocation") then

		IsSpawn = true

	elseif Object:IsA("BasePart")
		or Object:IsA("Model") then

		local Lower =
			string.lower(
				Object.Name
			)

		if string.find(
			Lower,
			"spawn",
			1,
			true
		)
		or string.find(
			Lower,
			"spawner",
			1,
			true
		) then

			IsSpawn = true

		end
	end

	if IsSpawn then

		local Position

		if Object:IsA("BasePart") then

			Position =
				Object.Position

		elseif Object:IsA("Model") then

			local Root =
				Object.PrimaryPart
				or Object:FindFirstChildWhichIsA(
					"BasePart",
					true
				)

			if Root then
				Position =
					Root.Position
			end
		end

		if Position then

			local Key =
				math.floor(Position.X)
				.. ":"
				.. math.floor(Position.Y)
				.. ":"
				.. math.floor(Position.Z)

			if not UsedPositions[Key] then

				UsedPositions[Key] = true

				local Name =
					Object.Name

				if Name == ""
					or Name == "Spawn"
					or Name == "SpawnLocation" then

					Name =
						"Spawn Humans"

				end

				table.insert(
					SpawnNames,
					Name
				)

				table.insert(
					SpawnObjects,
					Object
				)

			end
		end
	end
end

if #SpawnNames > 0 then

	CreateDropdown(
		TeleportPage,
		"Teleport",
		SpawnNames,
		SpawnNames[1],
		function(Value)

			for Index, Name in ipairs(
				SpawnNames
			) do

				if Name == Value then

					local Object =
						SpawnObjects[Index]

					if Object then

						local Root =
							GetRoot()

						if Root then

							local Target

							if Object:IsA(
								"BasePart"
							) then

								Target = Object

							elseif Object:IsA(
								"Model"
							) then

								Target =
									Object.PrimaryPart
									or Object:FindFirstChildWhichIsA(
										"BasePart",
										true
									)

							end

							if Target then

								Root.CFrame =
									Target.CFrame
									+ Vector3.new(
										0,3,0
									)

							end
						end
					end

					break
				end
			end
		end
	)

else

	CreateBox(
		TeleportPage,
		"Teleport: No spawns found"
	)

end

--========================================================--
-- SETTINGS
--========================================================--

CreateSection(
	SettingsPage,
	"Settings"
)

local function CreateNumberSetting(
	Page,
	Text,
	CurrentValue,
	Callback
)

	local Container =
		Instance.new("Frame")

	Container.Size =
		UDim2.new(1,-4,0,37)

	Container.BackgroundColor3 =
		Color3.fromRGB(24,24,28)

	Container.BorderSizePixel = 0
	Container.Parent = Page

	local Corner =
		Instance.new("UICorner")

	Corner.CornerRadius =
		UDim.new(0,5)

	Corner.Parent = Container

	local Label =
		Instance.new("TextLabel")

	Label.Size =
		UDim2.new(1,-115,1,0)

	Label.Position =
		UDim2.new(0,11,0,0)

	Label.BackgroundTransparency = 1
	Label.Text = Text

	Label.TextColor3 =
		Color3.fromRGB(190,190,195)

	Label.TextSize = 11
	Label.Font =
		Enum.Font.GothamMedium

	Label.TextXAlignment =
		Enum.TextXAlignment.Left

	Label.Parent = Container

	local Input =
		Instance.new("TextBox")

	Input.Size =
		UDim2.new(0,90,0,27)

	Input.Position =
		UDim2.new(1,-100,0,5)

	Input.BackgroundColor3 =
		Color3.fromRGB(17,17,20)

	Input.BorderSizePixel = 0
	Input.Text =
		tostring(CurrentValue)

	Input.TextColor3 =
		Color3.fromRGB(45,255,120)

	Input.TextSize = 11
	Input.Font =
		Enum.Font.GothamMedium

	Input.ClearTextOnFocus = false
	Input.Parent = Container

	local InputCorner =
		Instance.new("UICorner")

	InputCorner.CornerRadius =
		UDim.new(0,4)

	InputCorner.Parent = Input

	Input.FocusLost:Connect(function()

		local Number =
			tonumber(Input.Text)

		if Number
			and Number >= 0 then

			CurrentValue = Number

			Callback(Number)

			Input.Text =
				tostring(Number)

		else

			Input.Text =
				tostring(CurrentValue)

		end
	end)

	return Input
end

CreateNumberSetting(
	SettingsPage,
	"M1 Cooldown",
	M1Interval,
	function(Value)
		M1Interval = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Blood Drink Cooldown",
	BloodDrinkInterval,
	function(Value)
		BloodDrinkInterval = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Incendia Cooldown",
	IncendiaInterval,
	function(Value)
		IncendiaInterval = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Pain Cooldown",
	PainInterval,
	function(Value)
		PainInterval = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Quest Cooldown",
	QuestInterval,
	function(Value)
		QuestInterval = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Farm Distance",
	FarmDistance,
	function(Value)
		FarmDistance = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Nearby Money Radius",
	MoneyNearbyRadius,
	function(Value)
		MoneyNearbyRadius = Value
	end
)

CreateNumberSetting(
	SettingsPage,
	"Medium Money Radius",
	MoneyMediumRadius,
	function(Value)
		MoneyMediumRadius = Value
	end
)

--========================================================--
-- AUTO EXECUTE
--========================================================--

CreateToggle(
	SettingsPage,
	"Auto Execute",
	AutoExecuteEnabled,
	function(State)

		AutoExecuteEnabled = State

		getgenv().Config.AutoExecute =
			State

		if State then
			SetupAutoExecute()
		end

		SaveConfig()

	end
)

--========================================================--
-- ANTI AFK
--========================================================--

CreateToggle(
	SettingsPage,
	"Anti Afk",
	AntiAfkEnabled,
	function(State)

		AntiAfkEnabled =
			State

		getgenv().Config.AntiAfk =
			State

		if State then

			StartAntiAfk()

		else

			StopAntiAfk()

		end

		SaveConfig()

	end
)

--========================================================--
-- SAVE
--========================================================--

CreateToggle(
	SettingsPage,
	"Save",
	SaveEnabled,
	function(State)

		SaveEnabled =
			State

		getgenv().Config.SaveEnabled =
			State

		if State then

			SaveConfig()

		else

			UpdateConfigTable()

			getgenv().Config.SaveEnabled =
				false

			SaveConfig()

		end
	end
)

CreateBox(
	SettingsPage,
	"Cooldowns are controlled by the player"
)

--========================================================--
-- PESQUISA
--========================================================--

Search.Changed:Connect(function(Property)

	if Property ~= "Text" then
		return
	end

	local Query =
		string.lower(
			Search.Text
		)

	for Name, Button in pairs(
		NavigationButtons
	) do

		if Query == "" then

			Button.Visible = true

		else

			Button.Visible =
				string.find(
					string.lower(Name),
					Query,
					1,
					true
				) ~= nil

		end
	end
end)

--========================================================--
-- AUTO FARM LOOP
--========================================================--

task.spawn(function()

	while Gui.Parent do

		task.wait(0.05)

		if AutoCollectMoneyEnabled then
			AutoCollectMoney()
		end

		if not AutoFarmEnabled then
			continue
		end

		if not SelectedNPC then
			continue
		end

		local Humanoid =
			GetHumanoid()

		if not Humanoid
			or Humanoid.Health <= 0 then

			continue
		end

		if not CurrentNPC
			or not CurrentNPC.Parent
			or not IsNPCAlive(CurrentNPC)
			or GetNPCCategory(
				CurrentNPC
			) ~= SelectedNPC then

			CurrentNPC =
				FindNPC(
					SelectedNPC
				)

		end

		if not CurrentNPC then
			continue
		end

		local NPCRoot =
			GetNPCRoot(
				CurrentNPC
			)

		if not NPCRoot then

			CurrentNPC = nil

			continue
		end

		LockPlayerToNPC(
			CurrentNPC
		)

		EquipFists()

		DoQuest()

		DoAttacks(
			CurrentNPC
		)

	end
end)

--========================================================--
-- RESPAWN
--========================================================--

Player.CharacterAdded:Connect(function()

	CurrentNPC = nil

	task.wait(1)

	if AutoFarmEnabled then
		EquipFists()
	end

	if AntiAfkEnabled then
		StartAntiAfk()
	end

end)

--========================================================--
-- ATUALIZAR NPCS
--========================================================--

task.spawn(function()

	while Gui.Parent do

		task.wait(5)

		if Gui.Parent then
			RefreshNPCList()
		end

	end
end)

--========================================================--
-- AUTO EXECUTE INICIAL
--========================================================--

if AutoExecuteEnabled then

	task.defer(function()
		SetupAutoExecute()
	end)

end

--========================================================--
-- INICIAR
--========================================================--

ShowPage("Home")

print(
	"[Crazy Hub] Carregado."
)

if AntiAfkEnabled then

	print(
		"[Crazy Hub] Anti Afk ativado."
	)

end

if ConfigFileExists then

	print(
		"[Crazy Hub] Configuração anterior restaurada."
	)

end
