#-- GardenScript.lua (Server Script)
--!strict

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local Players = game:GetService("Players")

-- Module for seed data
local SeedData = require(ReplicatedStorage:WaitForChild("SeedDataModule"))
-- Module for pet data
local PetData = require(ReplicatedStorage:WaitForChild("PetDataModule"))

-- RemoteEvents for client-server communication
local GiveSeedEvent = ReplicatedStorage:WaitForChild("GiveSeedEvent")
local PlantSeedEvent = ReplicatedStorage:WaitForChild("PlantSeedEvent")
local SpawnPetEvent = ReplicatedStorage:WaitForChild("SpawnPetEvent")

-- Function to give a seed to a player
local function giveSeed(player: Player, seedName: string)
    local seedInfo = SeedData.Seeds[seedName]
    if not seedInfo then
        warn("Attempted to give unknown seed: " .. seedName)
        return
    end

    local seedModel = ServerStorage:FindFirstChild(seedInfo.SeedModelName)
    if not seedModel then
        warn("Seed model not found in ServerStorage: " .. seedInfo.SeedModelName)
        return
    end

    local clonedSeed = seedModel:Clone()
    clonedSeed.Parent = player.Backpack -- Or player.StarterGear for persistent tools
    print(player.Name .. " received " .. seedName .. " seed.")
end

-- Function to handle planting a seed
local function plantSeed(player: Player, seedName: string, position: Vector3)
    local seedInfo = SeedData.Seeds[seedName]
    if not seedInfo then
        warn("Attempted to plant unknown seed: " .. seedName)
        return
    end

    local plantableModel = ServerStorage:FindFirstChild(seedInfo.PlantableModelName)
    if not plantableModel then
        warn("Plantable model not found in ServerStorage: " .. seedInfo.PlantableModelName)
        return
    end

    local clonedPlant = plantableModel:Clone()
    clonedPlant:SetPrimaryPartCFrame(CFrame.new(position))
    clonedPlant.Parent = workspace.Plants -- Assuming a \'Plants\' folder in Workspace

    -- Initialize growth stage
    local growthStageValue = Instance.new("IntValue")
    growthStageValue.Name = "GrowthStage"
    growthStageValue.Value = 0
    growthStageValue.Parent = clonedPlant

    -- Start growth process
    task.spawn(function()
        local currentPlant = clonedPlant
        for i = 1, #seedInfo.GrowthStages do
            task.wait(seedInfo.GrowthTime)
            growthStageValue.Value = i

            local nextStageModelName = seedInfo.GrowthStages[i]
            local nextStageModel = ServerStorage:FindFirstChild(nextStageModelName)

            if nextStageModel then
                local newPlant = nextStageModel:Clone()
                newPlant:SetPrimaryPartCFrame(currentPlant.PrimaryPart.CFrame)
                newPlant.Parent = currentPlant.Parent
                currentPlant:Destroy()
                currentPlant = newPlant
                growthStageValue.Parent = currentPlant -- Re-parent the IntValue
                print(seedName .. " grew to stage " .. i)
            else
                warn("Growth stage model not found: " .. nextStageModelName)
                break
            end
        end
        print(seedName .. " fully grown!")
    end)

    print(player.Name .. " planted " .. seedName .. " at " .. tostring(position))
end

-- Function to handle spawning a pet
local function spawnPet(player: Player, petName: string, petSize: number)
    local petInfo = PetData.Pets[petName]
    if not petInfo then
        warn("Attempted to spawn unknown pet: " .. petName)
        return
    end

    local petModel = ServerStorage:FindFirstChild(petInfo.ModelName)
    if not petModel then
        warn("Pet model not found in ServerStorage: " .. petInfo.ModelName)
        return
    end

    local clonedPet = petModel:Clone()
    clonedPet.Parent = workspace.Pets -- Assuming a \'Pets\' folder in Workspace

    -- Set pet size (assuming the model has a PrimaryPart or can be scaled)
    if clonedPet.PrimaryPart then
        clonedPet:ScaleTo(petSize) -- This scales the model relative to its original size
    else
        warn("Pet model " .. petName .. " does not have a PrimaryPart for scaling.")
    end

    -- Position the pet near the player
    local spawnPosition = player.Character and player.Character.HumanoidRootPart.Position + Vector3.new(math.random(-5,5), 0, math.random(-5,5)) or Vector3.new(0, 10, 0)
    if clonedPet.PrimaryPart then
        clonedPet:SetPrimaryPartCFrame(CFrame.new(spawnPosition))
    else
        clonedPet:SetPivot(CFrame.new(spawnPosition)) -- For models without PrimaryPart
    end

    print(player.Name .. " spawned " .. petName .. " with size " .. petSize .. " at " .. tostring(spawnPosition))

    -- Basic pet roaming behavior (server-side, for demonstration)
    task.spawn(function()
        while clonedPet and clonedPet.Parent do
            local randomOffset = Vector3.new(math.random(-10, 10), 0, math.random(-10, 10))
            local targetPosition = spawnPosition + randomOffset
            if clonedPet.PrimaryPart and clonedPet.PrimaryPart:FindFirstChildOfClass("Humanoid") then
                clonedPet.PrimaryPart:FindFirstChildOfClass("Humanoid"):MoveTo(targetPosition)
                clonedPet.PrimaryPart:FindFirstChildOfClass("Humanoid").MoveToFinished:Wait()
            end
            task.wait(math.random(5, 10)) -- Wait before moving again
        end
    end)
end

-- Connect RemoteEvent listeners
GiveSeedEvent.OnServerEvent:Connect(giveSeed)
PlantSeedEvent.OnServerEvent:Connect(plantSeed)
SpawnPetEvent.OnServerEvent:Connect(spawnPet)

-- Example: Give all players a carrot seed when they join
Players.PlayerAdded:Connect(function(player)
    task.wait(1) -- Give time for backpack to load
    giveSeed(player, "Carrot")
end)

print("GardenScript loaded.")


-- SeedDataModule.lua (Module Script)
--!strict

local SeedDataModule = {}

SeedDataModule.Seeds = {
    ["Carrot"] = {
        SeedModelName = "CarrotSeedModel",
        PlantableModelName = "CarrotPlantStage0",
        GrowthStages = {"CarrotPlantStage1", "CarrotPlantStage2", "CarrotPlantStage3"},
        GrowthTime = 5, -- Seconds per stage
        Yield = {Name = "Carrot", Quantity = 1}
    },
    ["Tomato"] = {
        SeedModelName = "TomatoSeedModel",
        PlantableModelName = "TomatoPlantStage0",
        GrowthStages = {"TomatoPlantStage1", "TomatoPlantStage2", "TomatoPlantStage3", "TomatoPlantStage4"},
        GrowthTime = 7, -- Seconds per stage
        Yield = {Name = "Tomato", Quantity = 2}
    },
    -- Add more seeds here
}

return SeedDataModule


-- PetDataModule.lua (Module Script)
--!strict

local PetDataModule = {}

PetDataModule.Pets = {
    ["Dog"] = {
        ModelName = "DogModel",
        DefaultSize = 1,
        -- Add more pet properties here like abilities, animations, etc.
    },
    ["Cat"] = {
        ModelName = "CatModel",
        DefaultSize = 0.8,
    },
    -- Add more pets here
}

return PetDataModule


-- PlantingClientScript.lua (Local Script)
--!strict

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")

local Player = Players.LocalPlayer
local Mouse = Player:GetMouse()

local PlantSeedEvent = ReplicatedStorage:WaitForChild("PlantSeedEvent")
local SeedData = require(ReplicatedStorage:WaitForChild("SeedDataModule"))

local currentSelectedSeed = nil

-- Function to get the seed name from a tool
local function getSeedNameFromTool(tool: Tool): string | nil
    -- Assuming the tool name is the seed name, or there\'s an attribute
    -- For now, let\'s assume the tool\'s name is the seed name (e.g., \"CarrotSeed\")
    -- You might want to add an attribute to the tool for more robust identification
    local seedName = tool.Name:gsub("Seed", "") -- Remove \"Seed\" from tool name
    if SeedData.Seeds[seedName] then
        return seedName
    end
    return nil
end

-- Listen for tool equipping/unequipping
Player.CharacterAdded:Connect(function(character)
    local humanoid = character:WaitForChild("Humanoid")
    humanoid.EquipTool:Connect(function(tool)
        currentSelectedSeed = getSeedNameFromTool(tool)
        if currentSelectedSeed then
            print("Selected seed for planting: " .. currentSelectedSeed)
        else
            print("Tool equipped is not a recognized seed.")
        end
    end)
    humanoid.UnequipTool:Connect(function(tool)
        currentSelectedSeed = nil
        print("No seed selected for planting.")
    end)
end)

-- Handle mouse clicks for planting
UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and not gameProcessedEvent then
        if currentSelectedSeed then
            local target = Mouse.Target
            local hitPos = Mouse.Hit.Position

            -- Basic validation: Check if target is terrain or a baseplate
            -- You\'ll want more sophisticated validation on the server side
            if target and (target:IsA("Terrain") or target.Name == "Baseplate") then
                PlantSeedEvent:FireServer(currentSelectedSeed, hitPos)
                print("Attempting to plant " .. currentSelectedSeed .. " at " .. tostring(hitPos))
            else
                warn("Invalid planting location.")
            end
        else
            print("No seed tool equipped.")
        end
    end
end)

print("PlantingClientScript loaded.")


-- MainSpawnerGUI.lua (Local Script)
--!strict

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

local SeedData = require(ReplicatedStorage:WaitForChild("SeedDataModule"))
local GiveSeedEvent = ReplicatedStorage:WaitForChild("GiveSeedEvent")

-- Pet Data Module (will be created separately)
local PetData = nil
pcall(function() PetData = require(ReplicatedStorage:WaitForChild("PetDataModule")) end)
local SpawnPetEvent = ReplicatedStorage:WaitForChild("SpawnPetEvent")

-- Create ScreenGui
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "MainSpawnerGUI"
ScreenGui.Parent = PlayerGui
ScreenGui.ResetOnSpawn = false -- Keep GUI visible after character respawns

-- Create Main Frame
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 400, 0, 500) -- Example size
MainFrame.Position = UDim2.new(0.5, -200, 0.5, -250) -- Center of screen
MainFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40) -- Dark background
MainFrame.BorderSizePixel = 0
MainFrame.Draggable = true -- Make the frame draggable
MainFrame.Parent = ScreenGui

-- Create Title Bar
local TitleBar = Instance.new("Frame")
TitleBar.Name = "TitleBar"
TitleBar.Size = UDim2.new(1, 0, 0, 30)
TitleBar.Position = UDim2.new(0, 0, 0, 0)
TitleBar.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
TitleBar.BorderSizePixel = 0
TitleBar.Parent = MainFrame

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Name = "TitleLabel"
TitleLabel.Size = UDim2.new(1, -30, 1, 0)
TitleLabel.Position = UDim2.new(0, 0, 0, 0)
TitleLabel.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "Dark Spawner"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.Font = Enum.Font.SourceSansBold
TitleLabel.TextSize = 20
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.TextScaled = false
TitleLabel.TextWrapped = false
TitleLabel.Parent = TitleBar

-- Create Close Button
local CloseButton = Instance.new("TextButton")
CloseButton.Name = "CloseButton"
CloseButton.Size = UDim2.new(0, 30, 1, 0)
CloseButton.Position = UDim2.new(1, -30, 0, 0)
CloseButton.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
CloseButton.Text = "X"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Font = Enum.Font.SourceSansBold
CloseButton.TextSize = 20
CloseButton.Parent = TitleBar

CloseButton.MouseButton1Click:Connect(function()
    ScreenGui.Enabled = false -- Hide the GUI
end)

-- Make the GUI draggable (already set MainFrame.Draggable = true, but this is for custom drag if needed)
local dragging = false
local dragOffset = Vector2.new(0, 0)

TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragOffset = input.Position - UDim2.new(MainFrame.Position.X.Scale, MainFrame.Position.X.Offset, MainFrame.Position.Y.Scale, MainFrame.Position.Y.Offset).Offset
        TitleBar.BackgroundTransparency = 0.5 -- Visual feedback for dragging
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement and dragging then
        MainFrame.Position = UDim2.new(0, input.Position.X - dragOffset.X, 0, input.Position.Y - dragOffset.Y)
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
        TitleBar.BackgroundTransparency = 0 -- Reset transparency
    end
end)

-- Example: Add an open button to show the GUI (for testing)
local OpenButton = Instance.new("TextButton")
OpenButton.Name = "OpenSpawnerButton"
OpenButton.Size = UDim2.new(0, 100, 0, 30)
OpenButton.Position = UDim2.new(0.1, 0, 0.1, 0)
OpenButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
OpenButton.Text = "Open Spawner"
OpenButton.TextColor3 = Color3.fromRGB(255, 255, 255)
OpenButton.Font = Enum.Font.SourceSansBold
OpenButton.TextSize = 16
OpenButton.Parent = PlayerGui -- Place it directly in PlayerGui for easy access

OpenButton.MouseButton1Click:Connect(function()
    ScreenGui.Enabled = true -- Show the GUI
end)

-- Tab Container
local TabContainer = Instance.new("Frame")
TabContainer.Name = "TabContainer"
TabContainer.Size = UDim2.new(1, 0, 0, 40)
TabContainer.Position = UDim2.new(0, 0, 0, 30)
TabContainer.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
TabContainer.BorderSizePixel = 0
TabContainer.Parent = MainFrame

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.FillDirection = Enum.FillDirection.Horizontal
UIListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
UIListLayout.Padding = UDim.new(0, 5)
UIListLayout.Parent = TabContainer

-- Seed Spawner Tab Button
local SeedSpawnerButton = Instance.new("TextButton")
SeedSpawnerButton.Name = "SeedSpawnerButton"
SeedSpawnerButton.Size = UDim2.new(0.3, 0, 1, 0)
SeedSpawnerButton.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
SeedSpawnerButton.Text = "Seed Spawner"
SeedSpawnerButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SeedSpawnerButton.Font = Enum.Font.SourceSansBold
SeedSpawnerButton.TextSize = 16
SeedSpawnerButton.Parent = TabContainer

-- Pet Spawner Tab Button
local PetSpawnerButton = Instance.new("TextButton")
PetSpawnerButton.Name = "PetSpawnerButton"
PetSpawnerButton.Size = UDim2.new(0.3, 0, 1, 0)
PetSpawnerButton.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
PetSpawnerButton.Text = "Pet Spawner"
PetSpawnerButton.TextColor3 = Color3.fromRGB(255, 255, 255)
PetSpawnerButton.Font = Enum.Font.SourceSansBold
PetSpawnerButton.TextSize = 16
PetSpawnerButton.Parent = TabContainer

-- Content Frames
local SeedSpawnerContent = Instance.new("Frame")
SeedSpawnerContent.Name = "SeedSpawnerContent"
SeedSpawnerContent.Size = UDim2.new(1, 0, 1, 0)
SeedSpawnerContent.Position = UDim2.new(0, 0, 0, 0)
SeedSpawnerContent.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
SeedSpawnerContent.BorderSizePixel = 0
SeedSpawnerContent.Parent = ContentFrame

local PetSpawnerContent = Instance.new("Frame")
PetSpawnerContent.Name = "PetSpawnerContent"
PetSpawnerContent.Size = UDim2.new(1, 0, 1, 0)
PetSpawnerContent.Position = UDim2.new(0, 0, 0, 0)
PetSpawnerContent.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
PetSpawnerContent.BorderSizePixel = 0
PetSpawnerContent.Parent = ContentFrame
PetSpawnerContent.Visible = false -- Initially hidden

-- Function to switch tabs
local function switchTab(tabName: string)
    SeedSpawnerContent.Visible = (tabName == "SeedSpawner")
    PetSpawnerContent.Visible = (tabName == "PetSpawner")

    -- Update button colors
    SeedSpawnerButton.BackgroundColor3 = (tabName == "SeedSpawner") and Color3.fromRGB(90, 90, 90) or Color3.fromRGB(70, 70, 70)
    PetSpawnerButton.BackgroundColor3 = (tabName == "PetSpawner") and Color3.fromRGB(90, 90, 90) or Color3.fromRGB(70, 70, 70)
end

-- Connect tab buttons
SeedSpawnerButton.MouseButton1Click:Connect(function()
    switchTab("SeedSpawner")
end)
PetSpawnerButton.MouseButton1Click:Connect(function()
    switchTab("PetSpawner")
end)

-- Set initial tab
switchTab("SeedSpawner")

-- Seed Spawner UI Elements
-- Dropdown for seeds
local SeedDropdown = Instance.new("Frame")
SeedDropdown.Name = "SeedDropdown"
SeedDropdown.Size = UDim2.new(0.9, 0, 0, 30)
SeedDropdown.Position = UDim2.new(0.05, 0, 0.05, 0)
SeedDropdown.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
SeedDropdown.BorderSizePixel = 0
SeedDropdown.Parent = SeedSpawnerContent

local SeedDropdownLabel = Instance.new("TextLabel")
SeedDropdownLabel.Name = "SeedDropdownLabel"
SeedDropdownLabel.Size = UDim2.new(0.8, 0, 1, 0)
SeedDropdownLabel.Position = UDim2.new(0, 0, 0, 0)
SeedDropdownLabel.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
SeedDropdownLabel.BackgroundTransparency = 1
SeedDropdownLabel.Text = "Select a Seed"
SeedDropdownLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
SeedDropdownLabel.Font = Enum.Font.SourceSans
SeedDropdownLabel.TextSize = 16
SeedDropdownLabel.TextXAlignment = Enum.TextXAlignment.Left
SeedDropdownLabel.Parent = SeedDropdown

local SeedDropdownArrow = Instance.new("TextButton")
SeedDropdownArrow.Name = "SeedDropdownArrow"
SeedDropdownArrow.Size = UDim2.new(0.2, 0, 1, 0)
SeedDropdownArrow.Position = UDim2.new(0.8, 0, 0, 0)
SeedDropdownArrow.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
SeedDropdownArrow.Text = "▼"
SeedDropdownArrow.TextColor3 = Color3.fromRGB(255, 255, 255)
SeedDropdownArrow.Font = Enum.Font.SourceSansBold
SeedDropdownArrow.TextSize = 16
SeedDropdownArrow.Parent = SeedDropdown

local SeedDropdownList = Instance.new("ScrollingFrame")
SeedDropdownList.Name = "SeedDropdownList"
SeedDropdownList.Size = UDim2.new(1, 0, 0, 150) -- Max height for scroll
SeedDropdownList.Position = UDim2.new(0, 0, 1, 0)
SeedDropdownList.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
SeedDropdownList.BorderSizePixel = 0
SeedDropdownList.CanvasSize = UDim2.new(0, 0, 0, 0) -- Will be updated dynamically
SeedDropdownList.ScrollBarImageColor3 = Color3.fromRGB(100, 100, 100)
SeedDropdownList.ScrollBarThickness = 8
SeedDropdownList.Parent = SeedDropdown
SeedDropdownList.Visible = false

local UIListLayout_Dropdown = Instance.new("UIListLayout")
UIListLayout_Dropdown.FillDirection = Enum.FillDirection.Vertical
UIListLayout_Dropdown.Padding = UDim.new(0, 2)
UIListLayout_Dropdown.Parent = SeedDropdownList

local currentSelectedSeedName = nil

-- Populate dropdown with seeds
for seedName, seedInfo in pairs(SeedData.Seeds) do
    local SeedOptionButton = Instance.new("TextButton")
    SeedOptionButton.Name = seedName .. "Option"
    SeedOptionButton.Size = UDim2.new(1, 0, 0, 25)
    SeedOptionButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    SeedOptionButton.Text = seedName
    SeedOptionButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    SeedOptionButton.Font = Enum.Font.SourceSans
    SeedOptionButton.TextSize = 16
    SeedOptionButton.TextXAlignment = Enum.TextXAlignment.Left
    SeedOptionButton.Parent = SeedDropdownList

    SeedOptionButton.MouseButton1Click:Connect(function()
        currentSelectedSeedName = seedName
        SeedDropdownLabel.Text = seedName
        SeedDropdownList.Visible = false
        -- Update seed model display (placeholder for now)
        print("Selected: " .. seedName)
    end)
end

-- Adjust CanvasSize of dropdown list
SeedDropdownList.CanvasSize = UDim2.new(0, 0, 0, UIListLayout_Dropdown.AbsoluteContentSize.Y)

SeedDropdownArrow.MouseButton1Click:Connect(function()
    SeedDropdownList.Visible = not SeedDropdownList.Visible
end)

-- Spawn Seed Button
local SpawnSeedButton = Instance.new("TextButton")
SpawnSeedButton.Name = "SpawnSeedButton"
SpawnSeedButton.Size = UDim2.new(0.4, 0, 0, 40)
SpawnSeedButton.Position = UDim2.new(0.3, 0, 0.2, 0)
SpawnSeedButton.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
SpawnSeedButton.Text = "Spawn Seed"
SpawnSeedButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SpawnSeedButton.Font = Enum.Font.SourceSansBold
SpawnSeedButton.TextSize = 18
SpawnSeedButton.Parent = SeedSpawnerContent

SpawnSeedButton.MouseButton1Click:Connect(function()
    if currentSelectedSeedName then
        GiveSeedEvent:FireServer(currentSelectedSeedName)
        print("Fired GiveSeedEvent for: " .. currentSelectedSeedName)
    else
        warn("No seed selected to spawn.")
    end
end)

-- Pet Spawner UI Elements
local PetDropdown = Instance.new("Frame")
PetDropdown.Name = "PetDropdown"
PetDropdown.Size = UDim2.new(0.9, 0, 0, 30)
PetDropdown.Position = UDim2.new(0.05, 0, 0.05, 0)
PetDropdown.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
PetDropdown.BorderSizePixel = 0
PetDropdown.Parent = PetSpawnerContent

local PetDropdownLabel = Instance.new("TextLabel")
PetDropdownLabel.Name = "PetDropdownLabel"
PetDropdownLabel.Size = UDim2.new(0.8, 0, 1, 0)
PetDropdownLabel.Position = UDim2.new(0, 0, 0, 0)
PetDropdownLabel.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
PetDropdownLabel.BackgroundTransparency = 1
PetDropdownLabel.Text = "Select a Pet"
PetDropdownLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
PetDropdownLabel.Font = Enum.Font.SourceSans
PetDropdownLabel.TextSize = 16
PetDropdownLabel.TextXAlignment = Enum.TextXAlignment.Left
PetDropdownLabel.Parent = PetDropdown

local PetDropdownArrow = Instance.new("TextButton")
PetDropdownArrow.Name = "PetDropdownArrow"
PetDropdownArrow.Size = UDim2.new(0.2, 0, 1, 0)
PetDropdownArrow.Position = UDim2.new(0.8, 0, 0, 0)
PetDropdownArrow.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
PetDropdownArrow.Text = "▼"
PetDropdownArrow.TextColor3 = Color3.fromRGB(255, 255, 255)
PetDropdownArrow.Font = Enum.Font.SourceSansBold
PetDropdownArrow.TextSize = 16
PetDropdownArrow.Parent = PetDropdown

local PetDropdownList = Instance.new("ScrollingFrame")
PetDropdownList.Name = "PetDropdownList"
PetDropdownList.Size = UDim2.new(1, 0, 0, 150) -- Max height for scroll
PetDropdownList.Position = UDim2.new(0, 0, 1, 0)
PetDropdownList.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
PetDropdownList.BorderSizePixel = 0
PetDropdownList.CanvasSize = UDim2.new(0, 0, 0, 0) -- Will be updated dynamically
PetDropdownList.ScrollBarImageColor3 = Color3.fromRGB(100, 100, 100)
PetDropdownList.ScrollBarThickness = 8
PetDropdownList.Parent = PetDropdown
PetDropdownList.Visible = false

local UIListLayout_PetDropdown = Instance.new("UIListLayout")
UIListLayout_PetDropdown.FillDirection = Enum.FillDirection.Vertical
UIListLayout_PetDropdown.Padding = UDim.new(0, 2)
UIListLayout_PetDropdown.Parent = PetDropdownList

local currentSelectedPetName = nil

-- Populate dropdown with pets (requires PetData module)
if PetData then
    for petName, petInfo in pairs(PetData.Pets) do
        local PetOptionButton = Instance.new("TextButton")
        PetOptionButton.Name = petName .. "Option"
        PetOptionButton.Size = UDim2.new(1, 0, 0, 25)
        PetOptionButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
        PetOptionButton.Text = petName
        PetOptionButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        PetOptionButton.Font = Enum.Font.SourceSans
        PetOptionButton.TextSize = 16
        PetOptionButton.TextXAlignment = Enum.TextXAlignment.Left
        PetOptionButton.Parent = PetDropdownList

        PetOptionButton.MouseButton1Click:Connect(function()
            currentSelectedPetName = petName
            PetDropdownLabel.Text = petName
            PetDropdownList.Visible = false
            print("Selected Pet: " .. petName)
        end)
    end
    -- Adjust CanvasSize of pet dropdown list
    PetDropdownList.CanvasSize = UDim2.new(0, 0, 0, UIListLayout_PetDropdown.AbsoluteContentSize.Y)
else
    warn("PetDataModule not found. Pet spawner functionality will be limited.")
end

PetDropdownArrow.MouseButton1Click:Connect(function()
    PetDropdownList.Visible = not PetDropdownList.Visible
end)

-- Pet Size Input
local PetSizeInput = Instance.new("TextBox")
PetSizeInput.Name = "PetSizeInput"
PetSizeInput.Size = UDim2.new(0.9, 0, 0, 30)
PetSizeInput.Position = UDim2.new(0.05, 0, 0.2, 0)
PetSizeInput.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
PetSizeInput.PlaceholderText = "Enter pet size (e.g., 1, 2, 5)"
PetSizeInput.Text = "1" -- Default size
PetSizeInput.TextColor3 = Color3.fromRGB(255, 255, 255)
PetSizeInput.Font = Enum.Font.SourceSans
PetSizeInput.TextSize = 16
PetSizeInput.Parent = PetSpawnerContent

-- Spawn Pet Button
local SpawnPetButton = Instance.new("TextButton")
SpawnPetButton.Name = "SpawnPetButton"
SpawnPetButton.Size = UDim2.new(0.4, 0, 0, 40)
SpawnPetButton.Position = UDim2.new(0.3, 0, 0.35, 0)
SpawnPetButton.BackgroundColor3 = Color3.fromRGB(0, 120, 0)
SpawnPetButton.Text = "Spawn Pet"
SpawnPetButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SpawnPetButton.Font = Enum.Font.SourceSansBold
SpawnPetButton.TextSize = 18
SpawnPetButton.Parent = PetSpawnerContent

SpawnPetButton.MouseButton1Click:Connect(function()
    if currentSelectedPetName then
        local petSize = tonumber(PetSizeInput.Text) or 1
        SpawnPetEvent:FireServer(currentSelectedPetName, petSize)
        print("Fired SpawnPetEvent for: " .. currentSelectedPetName .. " with size " .. petSize)
    else
        warn("No pet selected to spawn.")
    end
end)

print("MainSpawnerGUI loaded.")


