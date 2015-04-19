#Python Scraper for ESPN NBA Data


import requests
from bs4 import BeautifulSoup
import re #module to get html data
import csv


###############################################################################
# UTILITY FUNCTIONS
###############################################################################


def soup_link(url):
	"""
	Input:
		url: a string, a valid url
	Returns:
		a BeautifulSoup object representing the HTML content of the url
	"""
	response = requests.get(url)
	cont = response.content
	soup = BeautifulSoup(cont)
	return soup



def write_list_of_dicts(list_dicts, path):
    """
    Inputs:
        list_dicts: a list of dictionaries
        path: a string, the filename where the output is written
    Returns:
        nothing. A file is written.
    """
    with open(path, "w") as f_out:
        header = list_dicts[0].keys()
        writer = csv.DictWriter(f_out, delimiter= ",", fieldnames=header)
        writer.writeheader()
        
        for dd in list_dicts:
		writer.writerow(dd)																						
																								
																											

def is_timestamp(string):
	"""g
	Determine if the string represents a timestamp.
	Input:
		string: a string to be analyzed
	Returns:
		a boolean telling if the string is in timestamp format or not.
	"""
	pattern_string = "\d\d?:\d{2}"  # one or two digits : two digits
	pattern = re.compile(pattern_string)
	return bool(pattern.search(string))



def is_result(string):
	"""
	Determine if the string represents a current result in the match.
	Input:
		string: a string to be analyzed
	Returns:
		a boolean telling if the string is a result or not.
	"""
	pattern_string = "^\d\d?\d?-\d\d?\d?"
	pattern = re.compile(pattern_string)
	return bool(pattern.search(string))



def parse_distance(string):
	"""
	Get the distance from a report line. Example: an input "25-foot jumper"
	should result in an output 25 (an integer).
	Input:
		string: a string to be analyzed
	Returns:
		if the string contains distance information, return the distance as
		an integer. else return nothing ("return None")
	"""
	pattern_string = "\d\d?-foot" #1 or 2 digits, followed by '-foot'
	pattern = re.compile(pattern_string)
	dd = pattern.search(string)
	if bool(dd) == True:  
		dist = map(str, re.findall(r'\d\d?-foot', string)) #Here we subsect the matched pattern
		dist = ', '.join(dist) # Unlist it 
		dist = dist.replace("-foot", "") #remove the common trailing string
		return int(dist) #and return the integer of the valued
	else:
		return None




def parse_points_scored(string):
	"""
	Determine the number of points scored.
	Input:
		string: a string to be analyzed
	Returns:
		1, 2 or 3: how many points were scored
	"""
	if "free throw" in string: 
		return 1
	elif "three point" in string:
		return 3
	else:
		return 2
		


def parse_shot_success(string):
    """
    Determine if the shot was successful.
	Input:
		string: a string to be analyzed
	Returns:
		"scores" or "misses" or None (return None) based on
		the shot success.
		parse_shot_success
	
    """
    if "makes" in string:
        return "scores"
    elif "misses" in string:
        return "misses"
    elif "blocks" in string:
        return "misses"
    else:
        return None


def get_player_in_action(string, list_players):
	"""
	Determine which player was in action
	Inputs:
		string: a string to be analyzed
		list_players: a list of players among whom we are looking for the
			active player
	Returns:
		The player in action. If none of the players in list_players was
		in action, return "Not in team"
	"""
	for player in list_players:
		begin_pattern = re.compile("^"+player)
		if begin_pattern.search(string):
			return player
		if "blocks" in string:
			pattern = re.compile(player)
			trimmed_string = ' '.join(string.split(' ')[2:])
			if pattern.search(trimmed_string):
				return player
	return "Not in team"



def is_shot_type(string, shot_type):
	"""
	If the shot_type is in string, return 1, else 0.
	"""
	if shot_type in string:
		return 1
	else:
		return 0 
		


###############################################################################
# SCRAPING FUNCTIONS
###############################################################################


def get_game_ids(team_id, year):
	"""
	Determine the ids of the games played by a team in a given year.
	Also, determine if the game was played home or away.
	Inputs:
		team_id: string, a three-letter id of the team
		year: integer, a year
	Returns:
		a list of tuples. Each tuple has a game id in the first place and
		"home" or "away" in the second place.
	"""
	
	url = "http://espn.go.com/nba/team/schedule/_/name/%s/year/%d/" % (team_id, year)
	soup = soup_link(url)

	gameid_pattern = "/nba/recap"
	game_elements = soup.find_all('a', href=re.compile(gameid_pattern))

	list_game_ids = []
	for game in game_elements:
		list_game_ids = map(int, re.findall(r'\d\d\d\d+', str(game_elements)))
	#print list_game_ids


	#home or away
	home_away_pattern = re.compile("game-status")
	status_elements = soup.find_all('li', class_= home_away_pattern)


	list_statuses = []
	for s_el in status_elements:
		if s_el.find("@") != -1:
			list_statuses.append("away")
		elif s_el.find("vs") != -1:
			list_statuses.append("home")
			

	return zip(list_game_ids, list_statuses)




def get_players(team_id, year):
	"""
	Get a list of players in a team in a given year.
	Inputs:
		team_id: string, a three-letter id of the team
		year: integer, a year
	Returns:
		a list of strings that are the players' names
	"""
	
	url = "http://espn.go.com/nba/team/stats/_/name/%s/year/%d/" %(team_id, year)
	response = requests.get(url)
	html_text = response.content
	soup = BeautifulSoup(html_text)
	roster = soup.find_all('tr', class_=re.compile('player'))

	player_names = [player.get_text().split(",")[0] for player in roster]
	player_names = player_names[:len(player_names)/2]

	return player_names


def extract_gamelog_row(row, list_players):
	"""
	Extract information from a gamelog row.
	Inputs:
		row: a gamelog row (Beautiful Soup object)
		list_players: a list of players
	Returns:
		a dictionary containing information about the gamelog row
	"""
	info_dict = {}

	pattern_info_line = re.compile("(text-align:left;|text-align:center;)")
	pieces = row.find_all('td', style=pattern_info_line)
     

	# filter rows of timeout, end of quarter
	if len(row) < 3:
	    	return {}
	for piece in pieces:
		text = piece.get_text()
		# get rid of initial spaces
		text = text.lstrip()

		if is_timestamp(text):
			info_dict["time"] = text
		elif is_result(text):
			info_dict["result"] = text
			info_dict["away_score"] = int(text.split("-")[0])
			info_dict["home_score"] = int(text.split("-")[1])
		else:
			info_dict["text"] = text

			# home or away
			if info_dict.has_key("result"):
				info_dict["home_or_away"] = "home"
			else:
				info_dict["home_or_away"] = "away"

			info_dict["player"] = get_player_in_action(text, list_players)
			info_dict["shot_success"] = parse_shot_success(text)

			if info_dict["shot_success"] == "scores":
				info_dict["points_scored"] = parse_points_scored(text)
			else:
				info_dict["points_scored"] = 0

			info_dict["dunk"] =  is_shot_type(text, "dunk")
			info_dict["layup"] = is_shot_type(text, "layup")
			info_dict["free_throw"] = is_shot_type(text, "free_throw")

			# distance
			info_dict["distance"] = parse_distance(text)

			# certain shot types are assigned a fixed distance
			if not(info_dict["distance"]):
				if info_dict["dunk"]:
					info_dict["distance"] = 0
				if info_dict["layup"]:
					info_dict["distance"] = 1
				if "tip shot" in text:
					info_dict["distance"] = 1
				if "three point" in text:
					info_dict["distance"] = 23

	if info_dict["home_or_away"] == "home":
		info_dict["own_score"] = info_dict["home_score"]
		info_dict["opponent_score"] = info_dict["away_score"]
	else:
		info_dict["own_score"] = info_dict["away_score"]
		info_dict["opponent_score"] = info_dict["home_score"]

	return info_dict	



def filter_game_info(list_info_dicts, home_or_away):
	"""
	Filter gamelog information.
	Inputs:
		list_info_dicts: list of dictionaries that represent gamelog rows
		home_or_away: string, "home" or "away"
	Returns:
		a list of dicts that represent filtered gamelog information
	"""
	filtered_list = []
	for d in list_info_dicts:
		if d.get("home_or_away") != home_or_away:
			continue
		if d["free_throw"]:
			continue
		if d["shot_success"] is None:
			continue
		if "home_score" not in d:
			continue

		d = {k: d[k] for k in ["text",
							   "dunk",
							   "distance",
							   "game_id",
							   "points_scored",
							   "layup",
							   "player",
							   "shot_success",
							   "time",
							   "own_score",
							   "opponent_score",
							   "home_or_away"]}

		filtered_list.append(d)

	return filtered_list


def structure_gamelog_info(game_id,
						   home_or_away,
						   list_players):
	"""
	Parse information from gamelog.
	Inputs:
		game_id: string, the id of the game
		home_or_away: string, "home" or "away"
		list_players: a list of strings, the players of the team
	Returns:
		list of dicts that contain information from the gamelog's rows
	"""
	game_link = "http://espn.go.com/nba/playbyplay?gameId=%s&period=0" %(game_id)
	gamelog = soup_link(game_link) 

	pattern_info = re.compile("(even|odd)") 
	report_lines = gamelog.find_all('tr', class_=pattern_info) 

	list_info_dicts = []
	for row in report_lines:
		info_dict = extract_gamelog_row(row, list_players)
		info_dict["game_id"] = game_id
		list_info_dicts.append(info_dict.copy())

	list_info_dicts_filtered = filter_game_info(list_info_dicts, home_or_away)

	return list_info_dicts_filtered


def parse_all_games(team_id, year, outdir):
	"""
	Parse gamelog information from all games of a team in a year.
	Inputs:
		team_id: string, the id of the team
		year: integer, the year
		outdir: string, the folder where the output file gets written
	"""
	print("SCRAPING GAMES OF %s IN SEASON %d" % (team_id, year))

	# GAME IDS
	game_ids_statuses = get_game_ids(team_id, year)

	# PLAYERS
	player_names = get_players(team_id, year)

	all_games_info_dicts = []

	count = 1
	for game_id, home_or_away in game_ids_statuses:
		print "Scraping game %d of %d" % (count, len(game_ids_statuses))
		count += 1

		list_info_dicts = structure_gamelog_info(game_id,
												 home_or_away,
												 player_names)
		all_games_info_dicts = all_games_info_dicts + list_info_dicts

	filename = outdir + "/all_games_%s_%d.csv" % (team_id, year)
	write_list_of_dicts(all_games_info_dicts, filename)


###############################################################################
# RUN THE SCRAPING
###############################################################################

year = 2014
DATA_FOLDER = "  "
list_team_ids = ["por", "bos"]

for team_id in list_team_ids:
	parse_all_games(team_id, year, DATA_FOLDER)
	



