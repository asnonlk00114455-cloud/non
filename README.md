local HttpService = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- รองรับ Executor หลายตัว
local req = (syn and syn.request) or (http and http.request) or http_request or (fluxus and fluxus.request) or request

-- GLOBAL CONFIG
getgenv().Info = {
    EngineOptions = { "Stockfish 17", "Sunfish" },
    Connections = {}
}

getgenv().Settings = {
    AutoPlay = false,
    Engine = "Stockfish 17",
    MoveDelay = 0.5,        -- Delay หลังเดินหมาก
    PreMoveDelay = 0.1       -- Delay ก่อนเรียก AI
}

-- ฟังก์ชัน Disconnect connections
local function DisconnectAll(connections)
    for _, connection in pairs(connections) do
        if connection then
            connection:Disconnect()
        end
    end
end

-- ฟังก์ชัน BestMove (รองรับ Stockfish / Sunfish) + delay ก่อนคำนวณ
local function BestMove(engine)
    task.wait(getgenv().Settings.PreMoveDelay)
    local selected = engine or getgenv().Settings.Engine
    local tableset = ReplicatedStorage.InternalClientEvents.GetActiveTableset:Invoke()
    if not tableset then return nil end

    local FEN = tableset:WaitForChild("FEN").Value
    local res

    if selected == "Stockfish 17" then
        local ok, err = pcall(function()
            res = req({
                Url = "https://chess-api.com/v1",
                Method = "POST",
                Headers = { ["Content-Type"] = "application/json" },
                Body = HttpService:JSONEncode({ fen = FEN }),
            })
        end)
        if ok and res and res.Success then
            local data = HttpService:JSONDecode(res.Body)
            return data.from, data.to
        else
            warn("[Engine] Stockfish error:", err or res.StatusCode)
            return nil, nil
        end

    elseif selected == "Sunfish" then
        local ok, result = pcall(function()
            local module = require(game:GetService("Players").LocalPlayer.PlayerScripts.AI.Sunfish)
            return module:GetBestMove(FEN, 1000)
        end)
        if ok and result then
            return result
        else
            warn("[Engine] Sunfish failed:", result)
            return nil, nil
        end
    end
end

-- ฟังก์ชัน PlayMove + delay หลังเดินหมาก
local function PlayMove(engine)
    local from, to = BestMove(engine)
    if from and to then
        ReplicatedStorage.Chess.SubmitMove:InvokeServer(from .. to)
        task.wait(getgenv().Settings.MoveDelay)
        return true
    elseif from then
        ReplicatedStorage.Chess.SubmitMove:InvokeServer(from)
        task.wait(getgenv().Settings.MoveDelay)
        return true
    end
    return false
end

-- ฟังก์ชัน PlaySuccesfullMove + fallback
local function PlaySuccesfullMove()
    local success = PlayMove()
    if not success then
        -- ถ้า Stockfish พัง → ลอง Sunfish
        success = PlayMove("Sunfish")
        if success then
            print("[Engine] Stockfish fail → Used Sunfish")
        else
            warn("[Engine] Both engines failed!")
        end
    end
end

-- ฟังก์ชัน AutoPlay (เชื่อม Event)
local function AutoPlay()
    if not getgenv().Settings.AutoPlay then return end

    DisconnectAll(getgenv().Info.Connections)

    getgenv().Info.Connections["MoveReceived"] =
        ReplicatedStorage.Chess.MovePlayedRemoteEvent.OnClientEvent:Connect(function()
            task.wait(getgenv().Settings.PreMoveDelay)
            PlaySuccesfullMove()
        end)

    getgenv().Info.Connections["GameStart"] =
        ReplicatedStorage.Chess.StartGameEvent.OnClientEvent:Connect(function()
            task.wait(getgenv().Settings.PreMoveDelay)
            PlaySuccesfullMove()
        end)

    PlaySuccesfullMove()
end
