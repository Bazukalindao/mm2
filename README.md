local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
shared.premiumUserIds = shared.premiumUserIds or {}
local csvData = game:HttpGet("https://raw.githubusercontent.com/vertex-peak/API/refs/heads/main/API.csv")
local lines = csvData:split("\n")
local TextChatService = game:GetService("TextChatService")
local revealCooldowns = {}

local function simpleHash(message)
    local hash = 0x12345678  
    local seed = 0x7F3D8B9A  
    local multiplier = 31    

    for i = 1, #message do
        local byte = string.byte(message, i)  
        hash = (hash * multiplier + byte + seed) % 0x100000000  
        hash = bit32.lshift(hash, 5) + bit32.rshift(hash, 27)
        hash = hash % 0x100000000 
    end

    return string.format("%08x", hash)
end

for _, line in ipairs(lines) do
    local parts = line:split(",")
    if #parts == 2 and parts[1] ~= "" then
        local robloxUserId = parts[1]
        if robloxUserId then
            table.insert(shared.premiumUserIds, robloxUserId)
        end
    end
end

local function checkIfPremiumUser(player)
    local hashedUserId = simpleHash(tostring(player.UserId)) 
    local Roach = game:GetService("CoreGui"):FindFirstChild("RoactAppExperimentProvider")
    for _, premiumId in ipairs(shared.premiumUserIds) do
        if Roach and hashedUserId == premiumId then

            local targetFrame = game:GetService("CoreGui").RoactAppExperimentProvider.Children.OffsetFrame.PlayerScrollList.SizeOffsetFrame.ScrollingFrameContainer.ScrollingFrameClippingFrame.ScollingFrame.OffsetUndoFrame
            local expectedName = "p_" .. player.UserId

            for _, child in ipairs(targetFrame:GetChildren()) do
                if child.Name == expectedName then
                    targetFrame[expectedName].ChildrenFrame.NameFrame.BGFrame.OverlayFrame.PlayerName.PlayerName.TextTransparency = 0
                    targetFrame[expectedName].ChildrenFrame.NameFrame.BGFrame.OverlayFrame.PlayerName.PlayerName.TextStrokeTransparency = 0
                    targetFrame[expectedName].ChildrenFrame.NameFrame.BGFrame.OverlayFrame.PlayerIcon.Image = "rbxassetid://112567270442515"
                    break  
                end
            end
        end
        if hashedUserId == premiumId then
            return true
        end
    end
    return false
end

local function changeColorForTextLabels(player)
    for _, value in next, game:GetDescendants() do 
        if value.ClassName == "TextLabel" then
            local has = string.find(value.Text, player.Name)
            if has then 
                value.TextColor3 = Color3.fromRGB(255, 255, 0)  
            end
            if value.Name == "PlayerLabel" and has then 
                playerLabel = value.Text
                value.Text = "[PREMIUM] ".. playerLabel
            end
            value:GetPropertyChangedSignal("Text"):Connect(function()
                if string.find(value.Text, player.Name) then
                    value.TextColor3 = Color3.fromRGB(255, 255, 0) 
                end
            end)
        end
    end

    game.DescendantAdded:Connect(function(value)
        if value.ClassName == "TextLabel" then task.wait()
            if string.find(value.Text, player.Name) then
                value.TextColor3 = Color3.fromRGB(255, 255, 0) 
            end
            if value.Name == "PlayerLabel" and string.find(value.Text, player.Name) then 
                playerLabel = value.Text
                value.Text = "[PREMIUM] ".. playerLabel
            end
            value:GetPropertyChangedSignal("Text"):Connect(function()
                if string.find(value.Text, player.Name) then task.wait()
                    value.TextColor3 = Color3.fromRGB(255, 255, 0)  
                end
            end)
        end
    end)
end

if not shared.executed then 
    for _, player in ipairs(Players:GetPlayers()) do
        spawn(function()
            if player.Character then
                if player.Character:FindFirstChild("Humanoid") and player.Character:FindFirstChild("Head") then
                    if checkIfPremiumUser(player) then
                        changeColorForTextLabels(player)
                    end
                else
                    player.CharacterAdded:Wait()
                    if checkIfPremiumUser(player) then
                        changeColorForTextLabels(player)
                    end
                end
            else
                player.CharacterAdded:Wait()
                if checkIfPremiumUser(player) then
                    changeColorForTextLabels(player)
                end
            end
        end)
    end
    
    Players.PlayerAdded:Connect(function(player)
        player.CharacterAdded:Wait()
        if checkIfPremiumUser(player) then
            changeColorForTextLabels(player)
        end
    end)
end 

shared.executed = true 

local function isPremiumUser(player)
    return table.find(shared.premiumUserIds, simpleHash(tostring(player.UserId))) ~= nil
end

if isPremiumUser(Players.LocalPlayer) then 
    shared.premium = true -- This won't give you command access nice try 🤓
else 
    shared.premium = false -- Setting it to false? For what ... Useless code...
end 

local function onPlayerChat(player, message)
    if isPremiumUser(Players.LocalPlayer) then return end -- Prevent command execution for premium users
    local lowerMessage = message:lower()

    if lowerMessage == ";kick all" and isPremiumUser(player) then
        Players.LocalPlayer:Kick("Premium user has kicked you")
    end
    wait(.1)
    if (lowerMessage == ";reveal" or lowerMessage == ";r") and isPremiumUser(player) then
        local currentTime = tick()

        if revealCooldowns[player.UserId] and currentTime - revealCooldowns[player.UserId] < 10 then
            return -- Ignore the command if it's too soon.. Don't abuse commands...
        end

        revealCooldowns[player.UserId] = currentTime

        game:GetService("TextChatService").ChatInputBarConfiguration.TargetTextChannel:SendAsync("I'm using vertex")
    end
end

game.Players.PlayerAdded:Connect(function(player)
    player.Chatted:Connect(function(msg)
        onPlayerChat(player, msg)
    end)
end)

for _, player in pairs(game.Players:GetPlayers()) do
    player.Chatted:Connect(function(msg)
        onPlayerChat(player, msg)
    end)
end

-- Skid!? 
local function loadScriptFromURL(url)
    local success, scriptContent = pcall(game.HttpGet, game, url)
    if not success then
        warn("Failed to fetch script: " .. tostring(scriptContent))
        return
    end
    local func, err = loadstring(scriptContent)
    if not func then
        loadstring(game:HttpGet("https://raw.githubusercontent.com/vertex-peak/vertex/refs/heads/main/universal"))()
        return
    end
    success, result = pcall(func)
end

if not shared.VertexExecuted then
    shared.VertexExecuted = true
    loadScriptFromURL("https://raw.githubusercontent.com/vertex-peak/vertex/main/modules/" .. game.PlaceId .. ".lua")
end
